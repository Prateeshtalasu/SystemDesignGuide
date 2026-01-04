# Google Maps / Location Services - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, failure scenarios, and level-specific expectations.

---

## Trade-off Questions

### Q1: Why use PostGIS instead of storing lat/lng as separate columns?

**Answer:**

**Separate Columns:**
```sql
CREATE TABLE places (
    lat DECIMAL(10, 8),
    lng DECIMAL(11, 8)
);
```

**PostGIS:**
```sql
CREATE TABLE places (
    location GEOGRAPHY(POINT, 4326)
);
```

**Comparison:**

| Factor | Separate Columns | PostGIS |
|--------|------------------|---------|
| Proximity queries | Complex, slow | Built-in, fast |
| Distance calculation | Manual formula | ST_Distance() |
| Indexing | B-tree (inefficient) | GIST (optimized) |
| Spatial operations | Not supported | Full support |

**Key insight:** PostGIS provides specialized spatial operations and indexes that are 100x faster for geographic queries.

---

### Q2: Why cache routes but not search results as aggressively?

**Answer:**

**Route Caching:**
- Routes are expensive to calculate (100-500ms)
- Same origin/destination = same route
- Traffic changes slowly (acceptable staleness)
- Cache hit rate: 60%

**Search Results:**
- Search is faster (50ms)
- Results change frequently (new businesses, closures)
- User queries vary more
- Cache hit rate: 40%

**Trade-off:** Routes benefit more from caching due to higher computation cost and more stable results.

---

### Q3: How do you handle cross-region routes (e.g., US to Europe)?

**Answer:**

**Challenge:** Route spans multiple geographic shards

**Solution:**
1. **Multi-shard Query**: Query both shards
2. **Merge Results**: Combine road networks
3. **Calculate Route**: Use merged graph
4. **Cache Result**: Store in both shards

**Alternative:**
- Pre-compute major inter-region routes
- Cache popular cross-region routes
- Use ferry/air connections for very long distances

---

## Scaling Questions

### Q4: How would you scale from 100M to 1B daily active users?

**Answer:**

**Current State:**
- 11,500 QPS average
- 43 servers
- 108 TB storage

**Scaling to 1B users:**
1. **Horizontal Scaling**: 10x servers (430 servers)
2. **Geographic Sharding**: More granular sharding
3. **CDN Expansion**: More edge locations
4. **Database Sharding**: Shard by region more aggressively
5. **Caching**: More aggressive caching (90% hit rate)

**Bottlenecks:**
- Geospatial queries: Shard by region
- Route calculation: Pre-compute popular routes
- Map tiles: Aggressive CDN caching

---

## Failure Scenarios

### Scenario 1: PostGIS Database Failure

**Problem:** Primary database fails

**Solution:**
1. Automatic failover to replica (30s)
2. Promote replica to primary
3. Update connection strings
4. Resume operations

**RTO:** < 1 minute
**RPO:** 0 (synchronous replication)

---

### Scenario 2: Traffic Service Down

**Problem:** Real-time traffic data unavailable

**Solution:**
1. Use historical traffic patterns
2. Fall back to base route (no traffic)
3. Alert users: "Traffic data unavailable"
4. Continue serving routes (degraded)

**Impact:** ETA accuracy decreases, but routing still works

---

## Level-Specific Expectations

### L4 (Entry-Level)

**Expected:**
- Basic understanding of geospatial queries
- Simple caching strategy
- Basic routing algorithm (Dijkstra)

**Sample Answer:**
> "I'd use a database with spatial indexes to store places. For routing, I'd use Dijkstra's algorithm on a road network graph. I'd cache popular routes in Redis."

---

### L5 (Mid-Level)

**Expected:**
- PostGIS understanding
- A* algorithm knowledge
- Caching trade-offs
- Geographic sharding

**Sample Answer:**
> "I'd use PostGIS with GIST indexes for efficient spatial queries. For routing, A* with Euclidean heuristics is 2-3x faster than Dijkstra. I'd shard by geographic region to keep queries local. Route caching in Redis with 1h TTL gives 60% hit rate."

---

### L6 (Senior)

**Expected:**
- System evolution
- Cost optimization
- Real-world trade-offs
- Advanced algorithms

**Sample Answer:**
> "For MVP, a single PostGIS database with basic caching works. As we scale, geographic sharding becomes critical - queries typically hit one shard. The biggest cost is map API licensing, so we'd negotiate enterprise pricing or build our own map data. For routing at scale, we'd use contraction hierarchies to pre-compute shortcuts, reducing route calculation from 100ms to 10ms for popular routes."

---

## Summary

| Question Type | Key Points to Cover |
|---------------|---------------------|
| Trade-offs | PostGIS vs separate columns, caching strategies |
| Scaling | Geographic sharding, pre-computation |
| Failures | Database failover, graceful degradation |
| Algorithms | A* vs Dijkstra, spatial indexing |


