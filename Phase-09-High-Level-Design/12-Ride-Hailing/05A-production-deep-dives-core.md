# Ride Hailing - Production Deep Dives (Core)

## Overview

This document covers the core production components: asynchronous messaging (Kafka for events), caching (Redis for real-time data), and search capabilities (geospatial queries). These are the fundamental building blocks that enable the ride-hailing system to meet its real-time requirements.

---

## 1. Async, Messaging & Event Flow (Kafka)

### A) CONCEPT: What is Kafka?

Apache Kafka is a distributed event streaming platform designed for high-throughput, fault-tolerant data pipelines. In a ride-hailing system, Kafka serves as the backbone for event-driven architecture, enabling services to communicate asynchronously.

**What problems does Kafka solve for ride-hailing?**

1. **Buffering**: Absorbs location update spikes during rush hours
2. **Decoupling**: Matching service doesn't need to wait for analytics
3. **Event sourcing**: Complete ride history for auditing and replay
4. **Real-time analytics**: Stream processing for surge pricing
5. **Durability**: Events persist even if consumers are temporarily down

**Core Kafka concepts:**

| Concept | Definition | Ride-Hailing Example |
|---------|------------|---------------------|
| **Topic** | A named stream of messages | `ride-events`, `location-updates` |
| **Partition** | A topic split for parallelism | Partitioned by `driver_id` |
| **Offset** | Unique ID for each message | Position in location stream |
| **Consumer Group** | Consumers sharing work | Analytics consumers |
| **Broker** | Kafka server storing messages | 6 brokers for HA |

### B) OUR USAGE: How We Use Kafka Here

**Why async vs sync for ride events?**

Synchronous processing would create bottlenecks:
- Location updates: 100K/second, can't block on each
- Ride events: Need audit trail without slowing ride flow
- Analytics: Surge calculation shouldn't delay matching

**Events in our system:**

| Event | Topic | Volume | Purpose |
|-------|-------|--------|---------|
| `LocationUpdated` | location-updates | 100K/sec | Driver tracking |
| `RideRequested` | ride-events | 300/sec | Audit, analytics |
| `RideStatusChanged` | ride-events | 1K/sec | State transitions |
| `PaymentProcessed` | payment-events | 300/sec | Financial audit |
| `SurgeCalculated` | surge-updates | 10/sec | Pricing updates |

**Topic/Queue Design:**

```
Topic: location-updates
Partitions: 48
Replication Factor: 3
Retention: 24 hours
Cleanup Policy: delete
Partition Key: driver_id

Topic: ride-events
Partitions: 24
Replication Factor: 3
Retention: 7 days
Cleanup Policy: delete
Partition Key: ride_id

Topic: payment-events
Partitions: 12
Replication Factor: 3
Retention: 30 days
Cleanup Policy: compact
Partition Key: ride_id
```

**Why 48 partitions for location-updates?**
- 100K updates/second peak
- Each partition handles ~3K messages/second efficiently
- 48 partitions allow 48 parallel consumers
- Room for growth without repartitioning

**Partition Key Choice:**

| Topic | Partition Key | Reasoning |
|-------|---------------|-----------|
| location-updates | driver_id | All updates for a driver go to same partition, preserving order |
| ride-events | ride_id | All events for a ride are ordered together |
| payment-events | ride_id | Payment events linked to ride |

### Hot Partition Mitigation

**Problem:** If a high-volume driver (e.g., very active in busy area) sends many location updates, one Kafka partition could become hot, creating a bottleneck for location processing.

**Example Scenario:**
```
Normal driver: 10 location updates/minute → Partition 15 (hash(driver_id) % partitions)
High-volume driver: 100 location updates/minute → Partition 15 (same partition!)
Result: Partition 15 receives 10x traffic, becomes bottleneck
Location analytics consumers for partition 15 become overwhelmed
```

**Detection:**
- Monitor partition lag per partition (Kafka metrics)
- Alert when partition lag > 2x average
- Track partition throughput metrics (updates/second per partition)
- Monitor consumer lag per partition
- Alert threshold: Lag > 2x average for 5 minutes

**Mitigation Strategies:**

1. **Partition Key Strategy:**
   - Partition by `driver_id` hash (distributes evenly across drivers)
   - For very active drivers, consider composite key (driver_id + time_bucket)
   - This distributes high-volume driver updates across multiple partitions

2. **Partition Rebalancing:**
   - If hot partition detected, increase partition count
   - Rebalance existing partitions (Kafka supports this)
   - Distribute hot driver updates across multiple partitions

3. **Consumer Scaling:**
   - Add more consumers for hot partitions
   - Auto-scale consumers based on partition lag
   - Each consumer handles subset of hot partition

4. **Rate Limiting:**
   - Rate limit per driver (prevent single driver from overwhelming)
   - Throttle high-volume drivers
   - Buffer location updates locally if rate limit exceeded

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
- Partition throughput (location updates/second per partition)
- Consumer lag per partition
- Alert threshold: Lag > 2x average for 5 minutes

**Consumer Group Design:**

```
Consumer Group: location-analytics
Consumers: 12 instances
Partitions per consumer: 4
Purpose: Calculate driver density, ETAs

Consumer Group: ride-event-processor
Consumers: 6 instances
Partitions per consumer: 4
Purpose: Update read models, send notifications

Consumer Group: surge-calculator
Consumers: 3 instances
Partitions per consumer: 8
Purpose: Real-time demand analysis
```

**Ordering Guarantees:**

- **Per-key ordering**: Guaranteed (all events for same driver/ride in order)
- **Global ordering**: Not guaranteed (not needed)
- **Exactly-once**: Achieved with idempotent consumers

**Offset Management:**

We use manual offset commits for reliability:

```java
@KafkaListener(topics = "ride-events", groupId = "ride-event-processor")
public void processRideEvent(
        List<RideEvent> events, 
        Acknowledgment ack) {
    try {
        for (RideEvent event : events) {
            // Process with idempotency check
            if (!eventRepository.exists(event.getEventId())) {
                processEvent(event);
                eventRepository.markProcessed(event.getEventId());
            }
        }
        ack.acknowledge();
    } catch (Exception e) {
        log.error("Failed to process ride events", e);
        // Don't acknowledge, will be reprocessed
    }
}
```

**Deduplication Strategy:**

Since we use at-least-once delivery, duplicates are possible:

```java
@Service
public class IdempotentEventProcessor {
    
    private final Set<String> processedEvents = ConcurrentHashMap.newKeySet();
    private final RedisTemplate<String, String> redis;
    
    public boolean shouldProcess(String eventId) {
        // Check local cache first
        if (processedEvents.contains(eventId)) {
            return false;
        }
        
        // Check Redis for distributed deduplication
        Boolean isNew = redis.opsForValue()
            .setIfAbsent("event:" + eventId, "1", Duration.ofHours(24));
        
        if (Boolean.TRUE.equals(isNew)) {
            processedEvents.add(eventId);
            return true;
        }
        return false;
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Event Flow: Driver Location Update**

```
Driver app sends location every 4 seconds
    ↓
WebSocket Gateway receives update
    ↓
Location Service processes:
    1. Validate coordinates
    2. Update Redis geospatial index (GEOADD)
    3. Update driver metadata hash (HSET)
    ↓
Publish to Kafka topic "location-updates":
    {
      "driverId": "driver_456",
      "latitude": 37.7749,
      "longitude": -122.4194,
      "heading": 45,
      "speed": 25,
      "timestamp": 1705312800000,
      "eventId": "loc_abc123"
    }
    ↓
Partition selected by hash(driver_456) % 48 = partition 12
    ↓
[ASYNC] Location Analytics Consumer:
    - Aggregates driver density per zone
    - Updates ETA models
    - Feeds surge pricing calculator
    ↓
[ASYNC] If driver has active ride:
    - Notify rider via WebSocket
    - Update ride ETA
```

**Event Flow: Ride Lifecycle**

```
Rider requests ride
    ↓
Trip Service creates ride (PostgreSQL)
    ↓
Publish RideRequested event:
    {
      "eventType": "RIDE_REQUESTED",
      "rideId": "ride_789",
      "riderId": "user_123",
      "pickup": {...},
      "dropoff": {...},
      "timestamp": 1705312800000
    }
    ↓
Matching Service finds driver
    ↓
Publish RideStatusChanged event:
    {
      "eventType": "DRIVER_ASSIGNED",
      "rideId": "ride_789",
      "driverId": "driver_456",
      "timestamp": 1705312830000
    }
    ↓
[ASYNC] Consumers process:
    - Analytics: Update ride metrics
    - Notification: Send push to rider
    - Audit: Store in event log
```

**Failure Scenarios:**

**Q: What if consumer crashes mid-batch?**

A: Offset is not committed, so the batch will be reprocessed. Our idempotent processing handles duplicates:

```java
// Each event has unique eventId
// Before processing, check if already processed
if (eventRepository.exists(event.getEventId())) {
    log.info("Skipping duplicate event: {}", event.getEventId());
    continue;
}
```

**Q: What if duplicate events occur?**

A: We handle duplicates at multiple levels:
1. Producer-side: Kafka idempotent producer (enable.idempotence=true)
2. Consumer-side: Deduplication by eventId in Redis
3. Database-side: Unique constraints prevent duplicate records

**Q: What if a partition becomes hot?**

A: Monitor and respond:
1. Alert on partition lag > 10,000 messages
2. Add consumers (up to partition count)
3. If persistent, consider repartitioning during low-traffic window

**Q: How does retry work?**

A: Producer retries with exponential backoff:

```java
config.put(ProducerConfig.RETRIES_CONFIG, 3);
config.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 100);
config.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 30000);
```

Consumer retries via not acknowledging:
- Message remains in partition
- Consumer group rebalances if consumer dies
- Dead letter queue for poison messages after 5 retries

**Q: What breaks first and what degrades gracefully?**

A: Order of failure:
1. **Location analytics lag**: Surge pricing delayed, but rides still work
2. **Notification consumer lag**: Push notifications delayed
3. **Kafka broker down**: Events queue in producer, rides continue with degraded analytics
4. **All Kafka down**: Location updates still work (Redis), events lost temporarily

### Event Schemas

```java
public abstract class RideEvent {
    private String eventId;
    private String eventType;
    private String rideId;
    private Instant timestamp;
}

public class RideRequestedEvent extends RideEvent {
    private String riderId;
    private Location pickup;
    private Location dropoff;
    private VehicleType vehicleType;
    private BigDecimal fareEstimate;
}

public class RideStatusChangedEvent extends RideEvent {
    private String previousStatus;
    private String newStatus;
    private String driverId;
    private Location location;
}

public class LocationUpdateEvent {
    private String eventId;
    private String driverId;
    private double latitude;
    private double longitude;
    private float heading;
    private float speed;
    private Instant timestamp;
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

        // Reliability for ride events
        config.put(ProducerConfig.ACKS_CONFIG, "all");  // Wait for all replicas
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        
        // Performance for location updates
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 32768);  // 32KB batches
        config.put(ProducerConfig.LINGER_MS_CONFIG, 10);  // Wait 10ms to batch
        config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");

        return new DefaultKafkaProducerFactory<>(config);
    }
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

---

## 2. Caching (Redis)

### A) CONCEPT: What is Redis and Caching Patterns?

Redis is an in-memory data structure store providing sub-millisecond latency. For ride-hailing, Redis serves multiple critical purposes:

1. **Geospatial index**: Finding nearby drivers in < 5ms
2. **Session store**: User authentication state
3. **Real-time state**: Active ride information
4. **Pub/Sub**: Cross-server WebSocket messaging

**Caching Patterns:**

| Pattern | Description | Use Case |
|---------|-------------|----------|
| **Cache-Aside** | App checks cache, then DB on miss | User profiles |
| **Write-Through** | Write to cache and DB together | Ride state |
| **Write-Behind** | Write to cache, async to DB | Location history |

**We use multiple patterns:**
- **Write-Through** for active rides (must be consistent)
- **Cache-Aside** for user profiles (can tolerate slight staleness)
- **Write-Only** for driver locations (Redis is source of truth for real-time)

### B) OUR USAGE: How We Use Redis Here

**Data Structures and Keys:**

```
1. DRIVER LOCATIONS (Geospatial Set)
   Key: drivers:locations
   Type: GEOADD / GEORADIUS
   Purpose: Find nearby drivers
   TTL: Members auto-expire via separate cleanup

2. DRIVER METADATA (Hash)
   Key: driver:{driver_id}
   Fields: status, vehicle_types, rating, last_update
   TTL: 30 seconds (offline if not refreshed)
   
3. ACTIVE RIDES (Hash)
   Key: ride:{ride_id}
   Fields: status, driver_id, rider_id, pickup, dropoff, created_at
   TTL: None (explicit delete on completion)
   
4. DRIVER CURRENT RIDE (String)
   Key: driver:{driver_id}:current_ride
   Value: ride_id
   TTL: None (explicit delete)
   
5. RIDER CURRENT RIDE (String)
   Key: rider:{rider_id}:current_ride
   Value: ride_id
   TTL: None (explicit delete)
   
6. SURGE PRICING (String)
   Key: surge:{zone_id}
   Value: multiplier (e.g., "1.5")
   TTL: 5 minutes
   
7. USER SESSIONS (Hash)
   Key: session:{session_id}
   Fields: user_id, user_type, device_id, created_at
   TTL: 30 minutes (sliding)
   
8. RATE LIMITING (Sorted Set)
   Key: ratelimit:{user_id}:{endpoint}
   Members: timestamps of requests
   TTL: 1 minute
```

**Who reads Redis?**
- Location Service: Driver locations for matching
- Trip Service: Active ride state
- Pricing Service: Surge multipliers
- API Gateway: Sessions, rate limits

**Who writes Redis?**
- Location Service: Driver locations, metadata
- Trip Service: Ride state
- Surge Calculator: Pricing data
- Auth Service: Sessions

**TTL Behavior:**

| Data | TTL | Reasoning |
|------|-----|-----------|
| Driver location | 30 sec | Stale location = offline driver |
| Active ride | None | Must persist until complete |
| Surge pricing | 5 min | Recalculated frequently |
| Session | 30 min sliding | Security + UX balance |

**Eviction Policy:**

```
maxmemory 8gb
maxmemory-policy volatile-lru
```

Why volatile-lru?
- Only evict keys with TTL set
- Active rides (no TTL) never evicted
- Driver locations (with TTL) can be evicted if needed

**Invalidation Strategy:**

On ride completion:
```java
public void completeRide(String rideId) {
    // 1. Update PostgreSQL
    rideRepository.updateStatus(rideId, "COMPLETED");
    
    // 2. Clean up Redis
    String driverId = redisTemplate.opsForHash().get("ride:" + rideId, "driver_id");
    String riderId = redisTemplate.opsForHash().get("ride:" + rideId, "rider_id");
    
    redisTemplate.delete("ride:" + rideId);
    redisTemplate.delete("driver:" + driverId + ":current_ride");
    redisTemplate.delete("rider:" + riderId + ":current_ride");
    
    // 3. Mark driver as available
    redisTemplate.opsForHash().put("driver:" + driverId, "status", "ONLINE");
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Cache HIT Path: Find Nearby Drivers**

```
Request: Find drivers within 5km of (37.7749, -122.4194)
    ↓
Matching Service receives request
    ↓
Redis: GEORADIUS drivers:locations -122.4194 37.7749 5 km 
       WITHCOORD WITHDIST COUNT 20 ASC
    ↓
Redis returns: [
    {driver_id: "d1", distance: 0.5km, coords: [...]},
    {driver_id: "d2", distance: 1.2km, coords: [...]},
    ...
]
    ↓
For each driver, get metadata:
Redis: HGETALL driver:d1
    ↓
Filter by: status=ONLINE, vehicle_type matches
    ↓
Return filtered list
    ↓
Total latency: ~5ms
```

**Cache MISS Path: Get User Profile**

```
Request: Get rider profile for user_123
    ↓
Trip Service checks Redis:
Redis: HGETALL user:user_123
    ↓
Redis returns: (nil) - MISS
    ↓
PostgreSQL: SELECT * FROM users WHERE id = 'user_123'
    ↓
DB returns user record
    ↓
Cache in Redis:
Redis: HMSET user:user_123 name "John" rating "4.8" ...
Redis: EXPIRE user:user_123 3600
    ↓
Return user profile
    ↓
Total latency: ~15ms (vs 5ms for cache hit)
```

**Write-Through: Update Ride Status**

```
Request: Update ride_789 status to IN_PROGRESS
    ↓
Trip Service:
    ↓
1. Update PostgreSQL (source of truth):
   UPDATE rides SET status = 'IN_PROGRESS', started_at = NOW()
   WHERE id = 'ride_789'
    ↓
2. Update Redis (real-time state):
   HSET ride:ride_789 status "IN_PROGRESS" started_at "2024-01-15T10:30:00Z"
    ↓
3. Publish event to Kafka
    ↓
4. Notify rider via WebSocket
    ↓
Total latency: ~20ms
```

**Failure Scenarios:**

**Q: What if Redis is down?**

A: Circuit breaker activates:

```java
@CircuitBreaker(name = "redis", fallbackMethod = "findDriversFromDB")
public List<NearbyDriver> findNearbyDrivers(Location pickup, double radius) {
    return redisGeoOperations.radius("drivers:locations", 
        new Circle(new Point(pickup.getLng(), pickup.getLat()), 
                   new Distance(radius, Metrics.KILOMETERS)));
}

public List<NearbyDriver> findDriversFromDB(Location pickup, double radius, Exception e) {
    log.warn("Redis unavailable, using database fallback", e);
    // PostGIS query - slower but functional
    return driverRepository.findNearby(pickup.getLat(), pickup.getLng(), radius);
}
```

Impact:
- Matching latency increases from 5ms to 50-100ms
- System still functional but degraded
- Location updates queue until Redis recovers

**Q: What happens to matching during Redis failure?**

A: Graceful degradation:
1. Circuit breaker opens after 50% failures
2. Fallback to PostgreSQL with PostGIS
3. Matching still works, just slower
4. Alert ops team for investigation

### Cache Stampede Prevention

**What is Cache Stampede?**

Cache stampede (also called "thundering herd") occurs when cached user profile or ride data expires and many requests simultaneously try to regenerate it, overwhelming the database.

**Scenario:**
```
T+0s:   Popular driver's profile cache expires
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
public class UserProfileCacheService {
    
    private final RedisTemplate<String, UserProfile> redis;
    private final DistributedLock lockService;
    private final UserRepository userRepository;
    
    public UserProfile getCachedProfile(String userId) {
        String key = "user:" + userId;
        
        // 1. Try cache first
        UserProfile cached = redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:user:" + userId;
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
            UserProfile profile = userRepository.findById(userId);
            
            if (profile != null) {
                // 5. Populate cache with TTL jitter (add ±10% random jitter)
                Duration baseTtl = Duration.ofHours(1);
                long jitter = (long)(baseTtl.getSeconds() * 0.1 * (Math.random() * 2 - 1));
                Duration ttl = baseTtl.plusSeconds(jitter);
                
                redis.opsForValue().set(key, profile, ttl);
            }
            
            return profile;
            
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

**Q: Why these TTLs?**

| Data | TTL | Reasoning |
|------|-----|-----------|
| Driver location | 30s | If no update in 30s, driver is offline |
| Surge pricing | 5min | Recalculated every minute, 5min buffer |
| Session | 30min | Balance security and UX |
| User profile | 1hr | Rarely changes, long cache |

### Redis Cluster Configuration

```yaml
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
        max-active: 200
        max-idle: 50
        min-idle: 20
```

### Geospatial Operations

```java
@Service
public class DriverLocationService {
    
    private final StringRedisTemplate redisTemplate;
    
    public void updateDriverLocation(String driverId, double lat, double lng) {
        // Add to geospatial index
        redisTemplate.opsForGeo().add(
            "drivers:locations",
            new Point(lng, lat),
            driverId
        );
        
        // Update driver metadata with TTL
        redisTemplate.opsForHash().putAll("driver:" + driverId, Map.of(
            "lat", String.valueOf(lat),
            "lng", String.valueOf(lng),
            "updated_at", Instant.now().toString()
        ));
        redisTemplate.expire("driver:" + driverId, Duration.ofSeconds(30));
    }
    
    public List<GeoResult<RedisGeoCommands.GeoLocation<String>>> findNearbyDrivers(
            double lat, double lng, double radiusKm, int limit) {
        
        return redisTemplate.opsForGeo().radius(
            "drivers:locations",
            new Circle(new Point(lng, lat), new Distance(radiusKm, Metrics.KILOMETERS)),
            RedisGeoCommands.GeoRadiusCommandArgs.newGeoRadiusArgs()
                .includeCoordinates()
                .includeDistance()
                .sortAscending()
                .limit(limit)
        ).getContent();
    }
}
```

---

## 3. Search (Geospatial Queries)

### A) CONCEPT: What is Geospatial Search?

Geospatial search finds data based on geographic location. For ride-hailing, the primary query is: "Find all drivers within X kilometers of this point."

**Traditional approaches and why they don't work:**

| Approach | Problem |
|----------|---------|
| SQL with lat/lng range | Not accurate (ignores Earth's curvature) |
| Full table scan | Too slow at scale |
| Simple indexing | Doesn't support radius queries |

**Geospatial indexing solutions:**

| Technology | How It Works | Use Case |
|------------|--------------|----------|
| **Geohash** | Encodes lat/lng into string, nearby points share prefix | Redis, Elasticsearch |
| **R-Tree** | Tree structure for spatial data | PostGIS, MongoDB |
| **Quadtree** | Divides space into quadrants recursively | Custom implementations |
| **H3** | Hexagonal hierarchical grid | Uber's solution |

### B) OUR USAGE: How We Use Geospatial Search Here

**Primary Technology: Redis Geospatial**

Redis uses geohash internally but provides a simple API:

```redis
# Add driver location
GEOADD drivers:locations -122.4194 37.7749 driver_456

# Find drivers within 5km
GEORADIUS drivers:locations -122.4194 37.7749 5 km WITHCOORD WITHDIST COUNT 20 ASC
```

**Why Redis Geospatial over PostGIS?**

| Aspect | Redis | PostGIS |
|--------|-------|---------|
| Latency | < 5ms | 20-50ms |
| Scale | 100K queries/sec | 5K queries/sec |
| Persistence | In-memory (can persist) | Disk-based |
| Updates | Instant | Requires index rebuild |

For real-time matching, Redis wins. We use PostGIS for:
- Historical analysis
- Backup/fallback
- Complex spatial queries (polygons, routes)

**Query Patterns:**

```java
@Service
public class GeoSearchService {
    
    // Primary: Find drivers for matching
    public List<NearbyDriver> findDriversNearPickup(Location pickup) {
        return findNearbyDrivers(pickup, 5.0, 20);  // 5km, top 20
    }
    
    // Find drivers in a specific zone (for surge calculation)
    public long countDriversInZone(GeoZone zone) {
        // Use PostGIS for polygon queries
        return driverRepository.countInPolygon(zone.getBoundary());
    }
    
    // Find rides near a location (for pooling)
    public List<Ride> findNearbyRides(Location point, double radiusKm) {
        // Active rides stored in Redis with location
        return findActiveRidesNear(point, radiusKm);
    }
}
```

**Index Design:**

```
Redis Key: drivers:locations
Type: Sorted Set with Geohash scores
Members: driver_id
Scores: 52-bit geohash of location

Operations:
- GEOADD: O(log(N)) - add/update driver
- GEORADIUS: O(N+log(M)) - find nearby (N=results, M=total)
- GEOPOS: O(1) - get driver location
- GEODIST: O(1) - distance between two members
```

### C) REAL STEP-BY-STEP SIMULATION

**Query Flow: Find Nearby Drivers**

```
Rider requests ride at (37.7749, -122.4194)
    ↓
Matching Service:
    ↓
1. Query Redis for nearby drivers:
   GEORADIUS drivers:locations -122.4194 37.7749 5 km 
   WITHCOORD WITHDIST COUNT 50 ASC
    ↓
2. Redis internally:
   - Convert query point to geohash
   - Find all geohash cells within radius
   - Return members in those cells
   - Calculate exact distance for each
   - Sort by distance, limit to 50
    ↓
3. Results: [
     {member: "d1", dist: 0.5, coord: [-122.4180, 37.7755]},
     {member: "d2", dist: 1.2, coord: [-122.4210, 37.7730]},
     ...
   ]
    ↓
4. For each driver, get metadata:
   HGETALL driver:d1
   → {status: "ONLINE", vehicle_types: "STANDARD,XL", rating: "4.9"}
    ↓
5. Filter:
   - status == "ONLINE"
   - vehicle_types contains requested type
   - No current ride
    ↓
6. Score and rank remaining drivers
    ↓
7. Return top 5 for ride request
    ↓
Total latency: ~10ms
```

**Handling Geospatial Edge Cases:**

**Q: What if no drivers within 5km?**

A: Expand search radius progressively:

```java
public List<NearbyDriver> findDriversWithExpansion(Location pickup) {
    double[] radii = {5.0, 10.0, 15.0, 20.0};
    
    for (double radius : radii) {
        List<NearbyDriver> drivers = findNearbyDrivers(pickup, radius, 10);
        if (!drivers.isEmpty()) {
            return drivers;
        }
    }
    
    return Collections.emptyList();  // No drivers available
}
```

**Q: How do we handle drivers at zone boundaries?**

A: Redis GEORADIUS handles this automatically by checking all nearby geohash cells.

**Q: What about drivers moving quickly?**

A: Location updates every 4 seconds keep index fresh. For matching:
1. Find candidates from index
2. Request current location from driver app
3. Verify still nearby before final match

### Geospatial Performance Optimization

```java
@Configuration
public class GeoSearchConfig {
    
    // Pipeline multiple geo queries
    public Map<String, List<NearbyDriver>> findDriversForMultiplePickups(
            List<Location> pickups) {
        
        return redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            for (Location pickup : pickups) {
                connection.geoCommands().geoRadius(
                    "drivers:locations".getBytes(),
                    new Circle(new Point(pickup.getLng(), pickup.getLat()),
                              new Distance(5, Metrics.KILOMETERS)),
                    GeoRadiusCommandArgs.newGeoRadiusArgs()
                        .includeCoordinates()
                        .includeDistance()
                        .limit(20)
                );
            }
            return null;
        });
    }
}
```

---

## Summary

| Component | Technology | Key Configuration |
|-----------|------------|-------------------|
| Event Streaming | Kafka | 48 partitions for locations, RF=3 |
| Real-time Cache | Redis Cluster | 8GB, 6 nodes, geospatial |
| Geospatial Search | Redis GEORADIUS | Sub-5ms queries |
| Fallback DB | PostgreSQL + PostGIS | For complex queries |

### Key Design Decisions

1. **Kafka for events**: Decouples services, enables replay, handles 100K+ events/sec
2. **Redis for real-time**: Sub-ms geospatial queries, shared state across services
3. **Write-through for rides**: Consistency between cache and DB
4. **TTL-based driver expiry**: Automatic offline detection
5. **Geohash-based search**: O(log N) updates, O(N) radius queries

---

## 4. Database Transactions Deep Dive

### A) CONCEPT: What are Database Transactions in Ride-Hailing Context?

Database transactions ensure ACID properties for critical operations like ride booking, payment processing, and driver assignment. In a ride-hailing system, transactions are essential for maintaining data consistency when multiple operations must succeed or fail together.

**What problems do transactions solve here?**

1. **Atomicity**: Ride booking must create ride record, update driver status, and reserve payment atomically
2. **Consistency**: Driver can't be assigned to multiple rides simultaneously
3. **Isolation**: Concurrent ride requests don't interfere with each other
4. **Durability**: Once committed, ride data persists even if system crashes

**Transaction Isolation Levels:**

| Level | Description | Use Case |
|-------|-------------|----------|
| READ COMMITTED | Default, prevents dirty reads | Most operations |
| REPEATABLE READ | Prevents non-repeatable reads | Ride status checks |
| SERIALIZABLE | Highest isolation, prevents phantom reads | Critical booking operations |

### B) OUR USAGE: How We Use Transactions Here

**Critical Transactional Operations:**

1. **Ride Booking Transaction**:
   - Create ride record
   - Update driver status (AVAILABLE → ASSIGNED)
   - Reserve payment
   - Create trip record
   - All must succeed or all rollback

2. **Payment Processing Transaction**:
   - Charge payment
   - Update ride status (CONFIRMED → PAID)
   - Update driver earnings
   - Create payment record

3. **Ride Completion Transaction**:
   - Update ride status (IN_PROGRESS → COMPLETED)
   - Calculate final fare
   - Process payment
   - Update driver location
   - Update driver status (ASSIGNED → AVAILABLE)

**Transaction Configuration:**

```java
@Transactional(
    isolation = Isolation.REPEATABLE_READ,
    propagation = Propagation.REQUIRED,
    timeout = 30,
    rollbackFor = {Exception.class}
)
public Ride createRide(CreateRideRequest request) {
    // Transactional operations
}
```

### C) REAL STEP-BY-STEP SIMULATION: Database Transaction Flow

**Normal Flow: Ride Booking Transaction**

```
Step 1: Begin Transaction
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: BEGIN TRANSACTION                                │
│ Isolation Level: REPEATABLE READ                             │
│ Transaction ID: tx_123456                                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Driver Availability
┌─────────────────────────────────────────────────────────────┐
│ SELECT status, current_ride_id FROM drivers                 │
│ WHERE id = 'driver_789'                                      │
│ FOR UPDATE  -- Row-level lock                                │
│                                                              │
│ Result: status = 'AVAILABLE', current_ride_id = NULL        │
│ Lock acquired on driver row                                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Create Ride Record
┌─────────────────────────────────────────────────────────────┐
│ INSERT INTO rides (id, rider_id, driver_id, status, ...)   │
│ VALUES ('ride_456', 'rider_123', 'driver_789', 'CONFIRMED') │
│                                                              │
│ Result: 1 row inserted                                      │
│ Ride ID: ride_456                                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Update Driver Status
┌─────────────────────────────────────────────────────────────┐
│ UPDATE drivers                                               │
│ SET status = 'ASSIGNED',                                     │
│     current_ride_id = 'ride_456',                           │
│     updated_at = NOW()                                       │
│ WHERE id = 'driver_789'                                      │
│                                                              │
│ Result: 1 row updated                                       │
│ Driver locked, status updated                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Reserve Payment
┌─────────────────────────────────────────────────────────────┐
│ INSERT INTO payments (id, ride_id, amount, status, ...)     │
│ VALUES ('pay_789', 'ride_456', 25.50, 'RESERVED', ...)      │
│                                                              │
│ Result: 1 row inserted                                       │
│ Payment reserved (not charged yet)                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 6: Commit Transaction
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: COMMIT TRANSACTION                              │
│                                                              │
│ All changes made permanent:                                 │
│ - Ride record created                                       │
│ - Driver status updated                                     │
│ - Payment reserved                                          │
│ - Locks released                                            │
│                                                              │
│ Total time: ~50ms                                            │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Deadlock Scenario**

```
Step 1: Two Concurrent Ride Requests
┌─────────────────────────────────────────────────────────────┐
│ Transaction A: Rider 1 requests ride, driver_789 selected │
│ Transaction B: Rider 2 requests ride, driver_789 selected  │
│ Both transactions start simultaneously                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Both Acquire Locks
┌─────────────────────────────────────────────────────────────┐
│ Transaction A: SELECT ... FOR UPDATE on driver_789         │
│                Lock acquired                                │
│                                                              │
│ Transaction B: SELECT ... FOR UPDATE on driver_789         │
│                Waiting for lock (blocked)                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Transaction A Tries to Update Another Resource
┌─────────────────────────────────────────────────────────────┐
│ Transaction A: UPDATE drivers SET ... WHERE id = 'driver_790'│
│                Needs lock on driver_790                      │
│                But driver_790 is locked by Transaction C    │
│                Transaction A blocked                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Deadlock Detected
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL Deadlock Detector:                               │
│ - Transaction A waiting for driver_790                      │
│ - Transaction C waiting for driver_789                      │
│ - Transaction A holds driver_789                            │
│ - Transaction C holds driver_790                            │
│                                                              │
│ Deadlock detected!                                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Deadlock Resolution
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: ROLLBACK TRANSACTION A                          │
│ Error: "deadlock detected"                                  │
│                                                              │
│ Application:                                                 │
│ 1. Catch DeadlockException                                  │
│ 2. Retry transaction with exponential backoff              │
│ 3. Select different driver if retry fails                  │
│                                                              │
│ Transaction B: Continues (lock released)                   │
│ Transaction C: Continues                                    │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Transaction Timeout**

```
Step 1: Long-Running Transaction
┌─────────────────────────────────────────────────────────────┐
│ BEGIN TRANSACTION (timeout: 30 seconds)                     │
│                                                              │
│ Step 1: Update ride status (5ms)                            │
│ Step 2: Process payment (external API call - 25 seconds)    │
│ Step 3: Update driver status (blocked waiting for lock)    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Timeout Occurs
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: Transaction timeout after 30 seconds            │
│ Error: "canceling statement due to statement timeout"      │
│                                                              │
│ Automatic ROLLBACK:                                          │
│ - All changes reverted                                      │
│ - Locks released                                            │
│ - Connection returned to pool                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Application Handling
┌─────────────────────────────────────────────────────────────┐
│ Application catches TransactionTimeoutException:            │
│                                                              │
│ 1. Log error with transaction context                      │
│ 2. Return 503 Service Unavailable to client                │
│ 3. Retry with shorter timeout or async processing          │
│ 4. Alert ops team if timeout rate > threshold              │
└─────────────────────────────────────────────────────────────┘
```

**Isolation Level Behavior: Concurrent Ride Requests**

```
Scenario: Two riders request rides, same driver available

Transaction A (Rider 1):                    Transaction B (Rider 2):
─────────────────────────────────────────────────────────────────
BEGIN TRANSACTION                          BEGIN TRANSACTION
                                           
SELECT status FROM drivers                  SELECT status FROM drivers
WHERE id = 'driver_789'                    WHERE id = 'driver_789'
FOR UPDATE                                  FOR UPDATE
                                           
Result: status = 'AVAILABLE'                Result: status = 'AVAILABLE'
                                           
UPDATE drivers                              UPDATE drivers
SET status = 'ASSIGNED'                     SET status = 'ASSIGNED'
WHERE id = 'driver_789'                    WHERE id = 'driver_789'
                                           
COMMIT                                     COMMIT
                                           
✅ Success: Driver assigned to Rider 1     ❌ Error: Constraint violation
                                           
                                           
With REPEATABLE READ isolation:            With READ COMMITTED:
- Transaction B sees 'AVAILABLE'           - Transaction B sees 'AVAILABLE'
  (snapshot from start)                      (latest committed value)
- Both try to update                        - Both try to update
- One succeeds, one fails                   - One succeeds, one fails
- Prevents double assignment                - Prevents double assignment
```

**Lock Contention: High-Concurrency Scenario**

```
Step 1: Surge Event - 100 Riders Request Rides Simultaneously
┌─────────────────────────────────────────────────────────────┐
│ 100 concurrent transactions trying to book rides           │
│ All targeting same 20 available drivers                   │
│                                                              │
│ Lock queue forms on driver rows                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Lock Acquisition
┌─────────────────────────────────────────────────────────────┐
│ First 20 transactions: Acquire locks immediately           │
│ Next 80 transactions: Wait in lock queue                  │
│                                                              │
│ PostgreSQL lock manager:                                   │
│ - Tracks waiting transactions                              │
│ - Orders by transaction start time (FIFO)                  │
│ - Prevents starvation                                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Lock Release and Cascade
┌─────────────────────────────────────────────────────────────┐
│ Transaction 1 commits:                                     │
│ - Lock on driver_1 released                                │
│ - Next waiting transaction acquires lock                   │
│                                                              │
│ This continues until all transactions complete             │
│                                                              │
│ Total time:                                                 │
│ - First 20: ~50ms each                                     │
│ - Next 80: 50ms + wait time (up to 2 seconds)             │
└─────────────────────────────────────────────────────────────┘
```

**Retry Logic for Failed Transactions**

```java
@Retryable(
    value = {DeadlockLoserDataAccessException.class, 
             TransactionTimeoutException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2)
)
@Transactional(isolation = Isolation.REPEATABLE_READ)
public Ride createRide(CreateRideRequest request) {
    // Transaction logic
    // Retries automatically on deadlock/timeout
}
```

**Recovery Behavior:**

```
Transaction Failure → Automatic Retry:
1. Deadlock detected → Retry with 100ms delay
2. Timeout → Retry with 200ms delay
3. Constraint violation → No retry (business logic error)
4. Max retries exceeded → Return error to client

Monitoring:
- Track retry rate (alert if > 5%)
- Track deadlock rate (alert if > 1%)
- Track timeout rate (alert if > 2%)
```

