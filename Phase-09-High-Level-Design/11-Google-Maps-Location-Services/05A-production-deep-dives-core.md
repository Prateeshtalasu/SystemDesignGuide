# Google Maps / Location Services - Production Deep Dives (Core)

## Overview

This document covers the core production components: caching strategy (Redis for search results and routes), geospatial indexing (QuadTree/R-tree), and routing algorithms (Dijkstra, A*).

---

## 1. Caching (Redis)

### A) CONCEPT: What is Redis Caching in Location Services Context?

Redis provides critical caching for location services:

1. **Search Result Cache**: Cache popular location searches
2. **Route Cache**: Cache frequently requested routes
3. **Geocoding Cache**: Cache address → coordinates mappings
4. **Map Tile Metadata**: Cache tile availability and metadata

### B) OUR USAGE: How We Use Redis Here

**Cache Types:**

| Cache | Key | TTL | Purpose |
|-------|-----|-----|---------|
| Search Results | `search:{query_hash}:{lat}:{lng}` | 1 hour | Popular searches |
| Routes | `route:{origin_hash}:{dest_hash}:{mode}` | 1 hour | Frequent routes |
| Geocoding | `geocode:{address_hash}` | 24 hours | Address lookups |
| Reverse Geocoding | `reverse:{lat}:{lng}` | 24 hours | Coordinate lookups |

**Search Result Caching:**

```java
@Service
public class SearchCacheService {
    
    private final RedisTemplate<String, SearchResults> redis;
    private static final Duration TTL = Duration.ofHours(1);
    
    public SearchResults getCachedResults(String query, double lat, double lng) {
        String key = buildKey(query, lat, lng);
        return redis.opsForValue().get(key);
    }
    
    public void cacheResults(String query, double lat, double lng, SearchResults results) {
        String key = buildKey(query, lat, lng);
        redis.opsForValue().set(key, results, TTL);
    }
    
    private String buildKey(String query, String lat, String lng) {
        String queryHash = DigestUtils.md5Hex(query);
        return String.format("search:%s:%s:%s", queryHash, lat, lng);
    }
}
```

### Cache Stampede Prevention

**What is Cache Stampede?**

Cache stampede (also called "thundering herd") occurs when a cached search result or route expires and many requests simultaneously try to regenerate it, overwhelming the search or routing service.

**Scenario:**
```
T+0s:   Popular route cache expires (e.g., "SF to LA")
T+0s:   1,000 users request same route simultaneously
T+0s:   All 1,000 requests see cache MISS
T+0s:   All 1,000 requests hit routing service simultaneously
T+1s:   Routing service overwhelmed, latency spikes
```

**Prevention Strategy: Distributed Lock + Double-Check Pattern**

This system uses distributed locking to ensure only one request regenerates the cache:

1. **Distributed locking**: Only one thread queries routing service on cache miss
2. **Double-check pattern**: Re-check cache after acquiring lock
3. **TTL jitter**: Add random jitter to TTL to prevent simultaneous expiration

**Implementation:**

```java
@Service
public class RouteCacheService {
    
    private final RedisTemplate<String, RouteResult> redis;
    private final DistributedLock distributedLock;
    private final RoutingService routingService;
    
    public RouteResult getCachedRoute(String origin, String destination, String mode) {
        String key = buildRouteKey(origin, destination, mode);
        
        // 1. Try cache first
        RouteResult cached = redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:" + key;
        boolean acquired = distributedLock.tryLock(lockKey, Duration.ofSeconds(5));
        
        if (!acquired) {
            // Another thread is computing, wait briefly
            try {
                Thread.sleep(50);
                cached = redis.opsForValue().get(key);
                if (cached != null) {
                    return cached;
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            // Fallback: Query routing service anyway (better than blocking)
            return routingService.calculateRoute(origin, destination, mode);
        }
        
        try {
            // 3. Double-check cache (might have been populated while acquiring lock)
            cached = redis.opsForValue().get(key);
            if (cached != null) {
                return cached;
            }
            
            // 4. Query routing service (only one thread does this)
            RouteResult result = routingService.calculateRoute(origin, destination, mode);
            
            // 5. Cache with TTL jitter (prevent simultaneous expiration)
            long baseTtl = 3600; // 1 hour
            long jitter = (long)(baseTtl * 0.1 * (Math.random() * 2 - 1));  // ±10%
            long finalTtl = baseTtl + jitter;
            
            redis.opsForValue().set(key, result, Duration.ofSeconds(finalTtl));
            
            return result;
            
        } finally {
            distributedLock.unlock(lockKey);
        }
    }
    
    private String buildRouteKey(String origin, String dest, String mode) {
        String originHash = DigestUtils.md5Hex(origin);
        String destHash = DigestUtils.md5Hex(dest);
        return String.format("route:%s:%s:%s", originHash, destHash, mode);
    }
}
```

**Same pattern applies to:**
- Search result caching
- Geocoding result caching
- Reverse geocoding caching

### C) REAL STEP-BY-STEP SIMULATION: Redis Technology-Level

**Normal Flow: Location Search with Cache**

```
Step 1: User Searches "coffee near me"
┌─────────────────────────────────────────────────────────────┐
│ Request: GET /v1/places/search?query=coffee&lat=37.7749&lng=-122.4194│
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Cache
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET search:abc123def:37.7749:-122.4194                │
│ Result: HIT (cached from previous search)                   │
│ Value: {                                                     │
│   "places": [...],                                          │
│   "cached_at": "2024-01-20T10:00:00Z"                      │
│ }                                                            │
│ Latency: ~1ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Return Cached Results
┌─────────────────────────────────────────────────────────────┐
│ Response: 200 OK                                             │
│ Body: Cached search results                                  │
│ Total latency: ~5ms (cache hit)                             │
└─────────────────────────────────────────────────────────────┘
```

**Cache Miss Flow:**

```
Step 1: Cache Miss
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET search:xyz789:37.7749:-122.4194 → MISS           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Query Database
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: SELECT * FROM places                            │
│              WHERE name ILIKE '%coffee%'                     │
│              AND ST_DWithin(location,                        │
│                  ST_MakePoint(-122.4194, 37.7749)::geography,│
│                  5000)                                       │
│ Latency: ~50ms                                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Cache Results
┌─────────────────────────────────────────────────────────────┐
│ Redis: SET search:xyz789:37.7749:-122.4194 {results} EX 3600│
│ Latency: ~1ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Return Results
┌─────────────────────────────────────────────────────────────┐
│ Response: 200 OK                                             │
│ Total latency: ~55ms (cache miss)                           │
└─────────────────────────────────────────────────────────────┘
```

**What if Redis is down?**
- Circuit breaker opens after 50% failure rate
- Fallback to direct database queries
- Latency increases from ~5ms to ~50ms
- System continues operating (graceful degradation)

---

## 2. Geospatial Indexing

### A) CONCEPT: What is Geospatial Indexing?

Geospatial indexing enables efficient queries on geographic data:
- **Proximity searches**: "Find all places within 1km"
- **Range queries**: "Find all places in this bounding box"
- **Nearest neighbor**: "Find the closest gas station"

Standard SQL indexes (B-tree) don't work for spatial queries because:
- They're designed for 1D data (numbers, strings)
- Geographic data is 2D (latitude, longitude)
- Distance calculations require special algorithms

### B) OUR USAGE: PostGIS GIST Indexes

**PostGIS Implementation:**

```sql
-- Create geospatial index
CREATE INDEX idx_places_location ON places 
USING GIST(location);

-- Proximity query
SELECT name, 
       ST_Distance(location, 
                   ST_MakePoint(-122.4194, 37.7749)::geography) as distance
FROM places
WHERE ST_DWithin(location,
                 ST_MakePoint(-122.4194, 37.7749)::geography,
                 1000)  -- 1km radius
ORDER BY distance
LIMIT 10;
```

**How GIST Index Works:**

1. **Spatial Partitioning**: Divides space into regions
2. **Minimum Bounding Rectangles (MBRs)**: Each region has an MBR
3. **Tree Structure**: Hierarchical organization
4. **Query Optimization**: Only searches relevant regions

**Performance:**
- Without index: Full table scan (O(n))
- With GIST index: Tree traversal (O(log n))
- 100M POIs: 1ms query vs 10 seconds

---

## 2.5. Database Operations (PostgreSQL + PostGIS) - Detailed Simulation

### A) CONCEPT: What is PostGIS Database Operations?

PostGIS extends PostgreSQL with geospatial capabilities. For location services, we use PostGIS for:
- Storing geographic coordinates (POINT, POLYGON)
- Spatial queries (proximity, distance, containment)
- Spatial indexing (GIST indexes for fast queries)

**Why PostGIS over separate lat/lng columns?**
- Specialized spatial operations (ST_Distance, ST_DWithin)
- Optimized spatial indexes (GIST)
- Handles coordinate system transformations
- 100x faster for geographic queries

### B) OUR USAGE: How We Use PostgreSQL + PostGIS

**Tables:**
```sql
-- Places table with PostGIS geography
CREATE TABLE places (
    place_id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(255),
    location GEOGRAPHY(POINT, 4326),  -- PostGIS geography type
    category VARCHAR(50),
    ...
);

-- GIST index for spatial queries
CREATE INDEX idx_places_location ON places USING GIST(location);
```

**Query Patterns:**
1. **Proximity Search**: Find places within radius
2. **Nearest Neighbor**: Find closest place
3. **Spatial Join**: Find places in polygon
4. **Distance Calculation**: Calculate distance between points

### C) REAL STEP-BY-STEP SIMULATION

**Proximity Search Flow:**

```
Step 1: User Searches "coffee near me"
┌─────────────────────────────────────────────────────────────┐
│ Request: GET /v1/places/search?query=coffee&lat=37.7749&lng=-122.4194│
│ User location: San Francisco (37.7749, -122.4194)         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Redis Cache (if applicable)
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET search:coffee:37.7749:-122.4194                │
│ Result: MISS (first search or cache expired)              │
│ Latency: ~1ms                                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Query PostgreSQL with PostGIS
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL Query:                                          │
│ SELECT name, category,                                      │
│        ST_Distance(location,                                │
│            ST_MakePoint(-122.4194, 37.7749)::geography)    │
│            AS distance                                      │
│ FROM places                                                 │
│ WHERE name ILIKE '%coffee%'                                │
│   AND ST_DWithin(location,                                 │
│       ST_MakePoint(-122.4194, 37.7749)::geography,        │
│       5000)  -- 5km radius                                 │
│ ORDER BY distance                                           │
│ LIMIT 20;                                                   │
│                                                             │
│ Execution Plan:                                             │
│ 1. Use GIST index (idx_places_location)                    │
│ 2. Filter by ST_DWithin (spatial filter)                   │
│ 3. Filter by name ILIKE (text filter)                      │
│ 4. Calculate distances                                      │
│ 5. Sort by distance                                         │
│                                                             │
│ Latency: ~15ms (with index)                                │
│ Without index: ~500ms (full table scan)                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Process Results
┌─────────────────────────────────────────────────────────────┐
│ Results: 15 coffee places found                             │
│ - Distance range: 0.2km to 4.8km                           │
│ - Categories: cafe, restaurant, bakery                     │
│ Processing: ~1ms                                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Cache Results
┌─────────────────────────────────────────────────────────────┐
│ Redis: SET search:coffee:37.7749:-122.4194 {results} EX 3600│
│ (Cache for 1 hour)                                         │
│ Latency: ~1ms                                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 6: Return Results
┌─────────────────────────────────────────────────────────────┐
│ Response: 200 OK                                            │
│ Body: 15 coffee places with distances                      │
│ Total latency: ~20ms (cache miss)                          │
└─────────────────────────────────────────────────────────────┘
```

**Transaction Flow: Adding New Place**

```
Step 1: Business Owner Adds New Place
┌─────────────────────────────────────────────────────────────┐
│ Request: POST /v1/places                                    │
│ Body: {                                                     │
│   "name": "New Coffee Shop",                                │
│   "lat": 37.7849,                                           │
│   "lng": -122.4094,                                         │
│   "category": "cafe"                                        │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Begin Transaction
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: BEGIN TRANSACTION                              │
│ (Start ACID transaction)                                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Insert Place with PostGIS
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: INSERT INTO places (                            │
│   place_id,                                                 │
│   name,                                                     │
│   location,  -- PostGIS GEOGRAPHY type                     │
│   category                                                  │
│ ) VALUES (                                                  │
│   'place_123',                                              │
│   'New Coffee Shop',                                        │
│   ST_MakePoint(-122.4094, 37.7849)::geography,             │
│   'cafe'                                                    │
│ );                                                          │
│                                                             │
│ PostGIS Operation:                                          │
│ - ST_MakePoint creates POINT from lat/lng                  │
│ - ::geography casts to geography type (WGS84)              │
│ - Index automatically updated (GIST)                       │
│                                                             │
│ Latency: ~5ms (with index update)                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Commit Transaction
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: COMMIT                                         │
│ (Transaction committed, data durable)                       │
│ Latency: ~2ms                                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Invalidate Cache
┌─────────────────────────────────────────────────────────────┐
│ Redis: DEL search:coffee:*  (pattern delete)              │
│ (Invalidate all coffee search caches)                      │
│ Note: In production, use more granular invalidation        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 6: Return Success
┌─────────────────────────────────────────────────────────────┐
│ Response: 201 Created                                       │
│ Body: {place_id: "place_123", ...}                        │
│ Total latency: ~10ms                                       │
└─────────────────────────────────────────────────────────────┘
```

**Replication Flow (Read Replicas):**

```
Step 1: Read Query (Load Balanced)
┌─────────────────────────────────────────────────────────────┐
│ Load Balancer routes read query                            │
│ - 70% to read replicas                                      │
│ - 30% to primary (if replicas busy)                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Query Read Replica
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL Read Replica:                                    │
│ SELECT * FROM places WHERE ...                             │
│                                                             │
│ Replication Lag: < 100ms                                   │
│ (Data may be slightly stale, acceptable for search)        │
│                                                             │
│ Latency: ~15ms (same as primary)                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Return Results
┌─────────────────────────────────────────────────────────────┐
│ Response: 200 OK                                            │
│ (Served from read replica, reduces primary load)           │
└─────────────────────────────────────────────────────────────┘
```

**What if Database is Down?**
- Circuit breaker opens after 50% failure rate
- Fallback to cached data in Redis
- Return stale results (acceptable for location search)
- Alert ops team for database issues

**What if GIST Index is Missing?**
- Query falls back to full table scan
- Latency increases from ~15ms to ~500ms
- Monitor slow query log
- Alert when queries > 100ms

---

## 3. Routing Algorithms

### A) CONCEPT: What is Route Calculation?

Route calculation finds the optimal path between two points on a road network. This is a graph problem where:
- **Nodes**: Intersections
- **Edges**: Road segments
- **Weights**: Distance, time, or cost

### B) OUR USAGE: Dijkstra and A* Algorithms

**Dijkstra Algorithm:**

```java
public class DijkstraRouter {
    
    public Route calculateRoute(Node start, Node end, RoadGraph graph) {
        Map<Node, Double> distances = new HashMap<>();
        Map<Node, Node> previous = new HashMap<>();
        PriorityQueue<Node> queue = new PriorityQueue<>(
            Comparator.comparing(distances::get)
        );
        
        distances.put(start, 0.0);
        queue.add(start);
        
        while (!queue.isEmpty()) {
            Node current = queue.poll();
            
            if (current.equals(end)) {
                return buildRoute(previous, start, end);
            }
            
            for (Edge edge : graph.getEdges(current)) {
                Node neighbor = edge.getTarget();
                double newDistance = distances.get(current) + edge.getWeight();
                
                if (newDistance < distances.getOrDefault(neighbor, Double.MAX_VALUE)) {
                    distances.put(neighbor, newDistance);
                    previous.put(neighbor, current);
                    queue.add(neighbor);
                }
            }
        }
        
        return null;  // No route found
    }
}
```

**A* Algorithm (with Heuristics):**

```java
public class AStarRouter {
    
    public Route calculateRoute(Node start, Node end, RoadGraph graph) {
        Map<Node, Double> gScore = new HashMap<>();  // Distance from start
        Map<Node, Double> fScore = new HashMap<>();  // gScore + heuristic
        Map<Node, Node> previous = new HashMap<>();
        PriorityQueue<Node> queue = new PriorityQueue<>(
            Comparator.comparing(fScore::get)
        );
        
        gScore.put(start, 0.0);
        fScore.put(start, heuristic(start, end));
        queue.add(start);
        
        while (!queue.isEmpty()) {
            Node current = queue.poll();
            
            if (current.equals(end)) {
                return buildRoute(previous, start, end);
            }
            
            for (Edge edge : graph.getEdges(current)) {
                Node neighbor = edge.getTarget();
                double newGScore = gScore.get(current) + edge.getWeight();
                
                if (newGScore < gScore.getOrDefault(neighbor, Double.MAX_VALUE)) {
                    gScore.put(neighbor, newGScore);
                    fScore.put(neighbor, newGScore + heuristic(neighbor, end));
                    previous.put(neighbor, current);
                    queue.add(neighbor);
                }
            }
        }
        
        return null;
    }
    
    private double heuristic(Node a, Node b) {
        // Euclidean distance (straight-line)
        double latDiff = a.getLat() - b.getLat();
        double lngDiff = a.getLng() - b.getLng();
        return Math.sqrt(latDiff * latDiff + lngDiff * lngDiff);
    }
}
```

**Why A* over Dijkstra?**
- A* uses heuristics to guide search
- Explores fewer nodes (faster)
- Still finds optimal route (if heuristic is admissible)
- 2-3x faster for typical routes

**Performance:**
- Dijkstra: 100ms for 10km route
- A*: 40ms for 10km route
- Both scale with graph size (millions of nodes)

---

## 4. Traffic Integration

### A) CONCEPT: Real-time Traffic Data

Traffic data comes from multiple sources:
1. **GPS devices**: Vehicles reporting speed/location
2. **Road sensors**: Embedded sensors measuring traffic
3. **Mobile apps**: Users contributing location data
4. **Historical patterns**: Time-of-day averages

### B) OUR USAGE: Traffic-Aware Routing

**Traffic Data Processing:**

```java
@Service
public class TrafficService {
    
    public void updateTrafficData(String segmentId, int speed, int speedLimit) {
        TrafficSegment segment = new TrafficSegment();
        segment.setSegmentId(segmentId);
        segment.setSpeed(speed);
        segment.setSpeedLimit(speedLimit);
        segment.setTimestamp(Instant.now());
        segment.setCongestionLevel(calculateCongestion(speed, speedLimit));
        
        // Store in TimescaleDB
        trafficRepository.save(segment);
        
        // Update in-memory cache for routing
        roadGraph.updateSegmentWeight(segmentId, calculateWeight(speed, speedLimit));
    }
    
    private CongestionLevel calculateCongestion(int speed, int speedLimit) {
        double ratio = (double) speed / speedLimit;
        if (ratio >= 0.9) return CongestionLevel.FREE_FLOW;
        if (ratio >= 0.7) return CongestionLevel.LIGHT;
        if (ratio >= 0.5) return CongestionLevel.MODERATE;
        if (ratio >= 0.3) return CongestionLevel.HEAVY;
        return CongestionLevel.SEVERE;
    }
}
```

**Traffic-Aware Route Calculation:**

```java
public Route calculateRouteWithTraffic(Node start, Node end, RoadGraph graph) {
    // Update edge weights based on current traffic
    for (Edge edge : graph.getEdges()) {
        TrafficData traffic = trafficService.getCurrentTraffic(edge.getSegmentId());
        if (traffic != null) {
            // Adjust weight: slower traffic = higher weight
            double trafficMultiplier = calculateTrafficMultiplier(traffic);
            edge.setWeight(edge.getBaseWeight() * trafficMultiplier);
        }
    }
    
    // Calculate route with updated weights
    return aStarRouter.calculateRoute(start, end, graph);
}
```

---

## Summary

| Component | Technology | Key Configuration |
|-----------|------------|-------------------|
| Search Cache | Redis | 1-hour TTL, 80% hit rate |
| Route Cache | Redis | 1-hour TTL, 60% hit rate |
| Geospatial Index | PostGIS GIST | Automatic spatial partitioning |
| Routing Algorithm | A* | Heuristic: Euclidean distance |
| Traffic Integration | TimescaleDB + In-memory | 30-second update latency |


