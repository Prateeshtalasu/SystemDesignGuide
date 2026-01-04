# Google Maps / Location Services - Data Model & Architecture

## Component Overview

Before looking at diagrams, let's understand each component and why it exists.

### Components Explained

| Component | Purpose | Why It Exists |
|-----------|---------|---------------|
| **API Gateway** | Entry point for all requests | Centralized routing, auth, rate limiting |
| **Search Service** | Location and place search | Fast geospatial queries with autocomplete |
| **Geocoding Service** | Address ↔ coordinates conversion | Users search by address, need coordinates |
| **Routing Engine** | Calculate optimal routes | Core navigation functionality |
| **Map Tile Service** | Serve map images | Visual map display requires pre-rendered tiles |
| **Traffic Service** | Real-time traffic data | ETA accuracy requires current conditions |
| **Geospatial Index** | Efficient proximity searches | Standard SQL can't handle spatial queries |
| **CDN** | Distribute map tiles globally | Low latency for global users |

---

## Database Choices

| Data Type | Database | Rationale |
|-----------|----------|-----------|
| POI Data | PostgreSQL + PostGIS | Geospatial queries, full-text search |
| Route Cache | Redis | Fast lookups, TTL support |
| Map Tiles | S3 + CDN | Blob storage, global distribution |
| Traffic Data | TimescaleDB | Time-series data, efficient aggregation |
| User Locations | PostgreSQL | Relational queries, privacy controls |

---

## Consistency Model

**CAP Theorem Tradeoff:**

We choose **Availability + Partition Tolerance (AP)**:
- **Availability**: Location services must always be available
- **Partition Tolerance**: System continues during network partitions
- **Consistency**: Eventual consistency acceptable for most operations

**Why AP over CP?**
- Search results can be slightly stale (acceptable)
- Route cache may be inconsistent across regions (acceptable)
- Traffic data is inherently eventually consistent
- Better to serve stale data than fail completely

**ACID vs BASE:**

**ACID (Strong Consistency) for:**
- User location updates (prevent duplicates)
- Route calculations (ensure accuracy)
- POI data updates (prevent conflicts)

**BASE (Eventual Consistency) for:**
- Traffic data (inherently real-time, acceptable staleness)
- Map tile updates (eventual propagation to CDN)
- Search index updates (eventual synchronization)

**Per-Operation Consistency Guarantees:**

| Operation | Consistency Level | Guarantee |
|-----------|------------------|-----------|
| Location search | Eventual | Results may be slightly stale |
| Route calculation | Strong | Route is accurate at calculation time |
| Traffic updates | Eventual | Data is 30-60 seconds old |
| Map tiles | Eventual | CDN propagation takes minutes |
| Geocoding | Strong | Address → coordinates is accurate |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        LOCATION SERVICES SYSTEM                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

                    ┌───────────────────┐
                    │   Mobile Apps     │
                    │   Web Clients    │
                    │   API Consumers  │
                    └────────┬──────────┘
                             │
                             ▼
                    ┌───────────────────┐
                    │   Load Balancer    │
                    │   (CloudFlare)     │
                    └────────┬──────────┘
                             │
                             ▼
                    ┌───────────────────┐
                    │   API Gateway     │
                    │  - Auth           │
                    │  - Rate Limiting  │
                    │  - Routing       │
                    └────────┬──────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   Search      │   │  Geocoding   │   │   Routing    │
│   Service     │   │   Service     │   │   Engine     │
│               │   │               │   │               │
│ - POI search  │   │ - Address →   │   │ - Route calc │
│ - Autocomplete│   │   Coordinates │   │ - ETA        │
│ - Nearby      │   │ - Reverse     │   │ - Alternatives│
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Map Tile     │   │   Traffic     │   │ Geospatial    │
│  Service      │   │   Service     │   │   Index        │
│               │   │               │   │               │
│ - Tile gen    │   │ - Real-time   │   │ - QuadTree    │
│ - CDN cache   │   │   updates     │   │ - R-tree      │
│ - Rendering   │   │ - Incidents   │   │ - Geohash     │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ PostgreSQL    │   │     Redis      │   │  TimescaleDB  │
│ + PostGIS     │   │   Cluster      │   │               │
│               │   │               │   │               │
│ - POI data    │   │ - Route cache │   │ - Traffic     │
│ - User locs   │   │ - Search cache│   │   segments    │
│ - Geospatial  │   │ - Session     │   │ - Incidents   │
└───────────────┘   └───────────────┘   └───────────────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                    ┌────────▼────────┐
                    │   S3 + CDN      │
                    │  - Map tiles    │
                    │  - Static assets│
                    └─────────────────┘
```

---

## Detailed Component Architecture

### Search Service Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              SEARCH SERVICE                                            │
└─────────────────────────────────────────────────────────────────────────────────────┘

                    ┌───────────────────┐
                    │  Search Request   │
                    │  "coffee near me" │
                    └────────┬──────────┘
                             │
                             ▼
                    ┌───────────────────┐
                    │  Query Parser     │
                    │  - Extract query  │
                    │  - Parse location │
                    │  - Parse filters  │
                    └────────┬──────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
                    ▼                 ▼
        ┌──────────────────┐  ┌──────────────────┐
        │  Text Search      │  │  Geospatial      │
        │  (Full-text)      │  │  Search          │
        │                   │  │                   │
        │ - PostgreSQL GIN  │  │ - PostGIS GIST    │
        │   index           │  │   index           │
        │ - Name matching   │  │ - Proximity       │
        └────────┬──────────┘  └────────┬──────────┘
                 │                      │
                 └──────────┬───────────┘
                            │
                            ▼
                    ┌───────────────────┐
                    │  Result Merger     │
                    │  - Combine results│
                    │  - Rank by score   │
                    │  - Apply filters  │
                    └────────┬──────────┘
                             │
                             ▼
                    ┌───────────────────┐
                    │  Cache Result     │
                    │  (Redis, 1h TTL)  │
                    └────────┬──────────┘
                             │
                             ▼
                    ┌───────────────────┐
                    │  Return Results   │
                    └───────────────────┘
```

---

### Routing Engine Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              ROUTING ENGINE                                           │
└─────────────────────────────────────────────────────────────────────────────────────┘

                    ┌───────────────────┐
                    │  Route Request   │
                    │  Origin → Dest    │
                    └────────┬──────────┘
                             │
                             ▼
                    ┌───────────────────┐
                    │  Check Cache       │
                    │  (Redis)           │
                    └────────┬──────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
                    ▼                 ▼
            ┌───────────┐    ┌──────────────┐
            │ Cache HIT │    │  Cache MISS  │
            │ Return    │    │  Calculate   │
            └───────────┘    └──────┬───────┘
                                    │
                                    ▼
                    ┌───────────────────────────┐
                    │  Load Road Network Graph  │
                    │  (In-memory)              │
                    └─────────────┬─────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
        ┌──────────────────┐      ┌──────────────────┐
        │  Dijkstra        │      │  A* Algorithm    │
        │  Algorithm       │      │  (with heuristics)│
        │                  │      │                  │
        │ - Shortest path  │      │ - Faster         │
        │ - Guaranteed     │      │ - Near-optimal   │
        └────────┬─────────┘      └────────┬─────────┘
                 │                         │
                 └──────────┬──────────────┘
                            │
                            ▼
                    ┌───────────────────┐
                    │  Apply Traffic    │
                    │  - Adjust weights │
                    │  - Recalculate    │
                    └────────┬──────────┘
                             │
                             ▼
                    ┌───────────────────┐
                    │  Generate Polyline │
                    │  - Encode route   │
                    │  - Calculate ETA  │
                    └────────┬──────────┘
                             │
                             ▼
                    ┌───────────────────┐
                    │  Cache Route      │
                    │  (Redis, 1h TTL)  │
                    └────────┬──────────┘
                             │
                             ▼
                    ┌───────────────────┐
                    │  Return Route     │
                    └───────────────────┘
```

---

## Geospatial Indexing Strategies

### QuadTree for Proximity Search

**What is QuadTree?**

A QuadTree is a tree data structure that recursively divides a 2D space into four quadrants. Each node represents a rectangular region, and child nodes represent sub-regions.

**Structure:**
```
                    ┌─────────────┐
                    │   Root      │
                    │  (Entire    │
                    │   World)    │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
   ┌─────────┐      ┌─────────┐      ┌─────────┐
   │  NW     │      │  NE     │      │  SW     │
   │ Quadrant│      │ Quadrant│      │ Quadrant│
   └─────────┘      └─────────┘      └─────────┘
```

**Implementation:**
```java
public class QuadTree {
    
    private static final int MAX_POINTS = 10;
    private static final int MAX_DEPTH = 20;
    
    private Rectangle bounds;
    private List<Point> points;
    private QuadTree[] children;
    private int depth;
    
    public List<Point> search(Rectangle query) {
        if (!bounds.intersects(query)) {
            return Collections.emptyList();
        }
        
        List<Point> results = new ArrayList<>();
        
        // Check points in this node
        for (Point point : points) {
            if (query.contains(point)) {
                results.add(point);
            }
        }
        
        // Search children
        if (children != null) {
            for (QuadTree child : children) {
                results.addAll(child.search(query));
            }
        }
        
        return results;
    }
}
```

**Why QuadTree?**
- O(log n) average search time
- Efficient for 2D spatial queries
- Good for evenly distributed points
- Simple to implement

**Limitations:**
- Degrades with clustered data
- Not optimal for high-dimensional data
- Rebalancing can be expensive

---

### R-tree for Complex Geometries

**What is R-tree?**

An R-tree is a tree data structure designed for indexing multi-dimensional information. It groups nearby objects and represents them with minimum bounding rectangles (MBRs).

**Structure:**
```
                    ┌─────────────┐
                    │   Root      │
                    │  [MBR1,MBR2]│
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
   ┌─────────┐      ┌─────────┐      ┌─────────┐
   │  Node1  │      │  Node2  │      │  Node3  │
   │ [MBR...]│      │ [MBR...]│      │ [MBR...]│
   └─────────┘      └─────────┘      └─────────┘
```

**Why R-tree?**
- Handles complex geometries (polygons, lines)
- Efficient for range queries
- Used by PostGIS internally
- Good for real-world geographic data

---

### Geohash for Simple Proximity

**What is Geohash?**

Geohash is a geocoding method that encodes a geographic location into a short string. Nearby locations share a common prefix.

**Example:**
```
Location: 37.7749, -122.4194 (San Francisco)
Geohash:  9q8yy

Nearby locations:
  9q8yy → 9q8yy (exact match)
  9q8yz → 9q8y (shared prefix)
  9q8yx → 9q8y (shared prefix)
```

**Implementation:**
```java
public class Geohash {
    
    public static String encode(double lat, double lng, int precision) {
        double[] latRange = {-90.0, 90.0};
        double[] lngRange = {-180.0, 180.0};
        
        StringBuilder hash = new StringBuilder();
        boolean even = true;
        int bit = 0;
        int ch = 0;
        
        while (hash.length() < precision) {
            if (even) {
                double mid = (lngRange[0] + lngRange[1]) / 2;
                if (lng >= mid) {
                    ch |= (1 << (4 - bit));
                    lngRange[0] = mid;
                } else {
                    lngRange[1] = mid;
                }
            } else {
                double mid = (latRange[0] + latRange[1]) / 2;
                if (lat >= mid) {
                    ch |= (1 << (4 - bit));
                    latRange[0] = mid;
                } else {
                    latRange[1] = mid;
                }
            }
            
            even = !even;
            if (bit < 4) {
                bit++;
            } else {
                hash.append(BASE32.charAt(ch));
                bit = 0;
                ch = 0;
            }
        }
        
        return hash.toString();
    }
}
```

**Why Geohash?**
- Simple string encoding
- Prefix matching for proximity
- Easy to shard by geohash prefix
- Good for caching strategies

---

## Sharding Strategy

### Geographic Sharding

**Shard by Region:**
```
Shard 1: North America (lat: 25-50, lng: -125 to -65)
Shard 2: Europe (lat: 35-70, lng: -10 to 40)
Shard 3: Asia (lat: 10-50, lng: 60-150)
Shard 4: South America (lat: -55 to 15, lng: -80 to -35)
...
```

**Benefits:**
- Queries typically hit one shard
- Data locality (users in same region)
- Easier to scale by region

**Challenges:**
- Cross-region queries (e.g., route from US to Europe)
- Uneven distribution (some regions have more data)
- Rebalancing when regions grow

**Implementation:**
```java
public class GeographicSharding {
    
    public int getShard(double lat, double lng) {
        // Determine region
        if (lat >= 25 && lat <= 50 && lng >= -125 && lng <= -65) {
            return 1;  // North America
        } else if (lat >= 35 && lat <= 70 && lng >= -10 && lng <= 40) {
            return 2;  // Europe
        }
        // ... more regions
        
        return 0;  // Default shard
    }
}
```

---

## Replication Strategy

### Read Replicas

**Configuration:**
- Primary: 1 per shard (writes)
- Replicas: 2-3 per shard (reads)

**Read Distribution:**
- 70% of reads → Replicas
- 30% of reads → Primary (for consistency)

**Benefits:**
- Scale read capacity
- Geographic distribution (replicas in different AZs)
- High availability (failover to replica)

---

## Summary

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| POI Storage | PostgreSQL + PostGIS | Geospatial queries, full-text search |
| Route Cache | Redis | Fast lookups, TTL support |
| Map Tiles | S3 + CDN | Global distribution, low latency |
| Traffic Data | TimescaleDB | Time-series optimization |
| Geospatial Index | PostGIS GIST | Efficient spatial queries |
| Sharding | Geographic | Data locality, query efficiency |
| Replication | Read replicas | Scale reads, high availability |


