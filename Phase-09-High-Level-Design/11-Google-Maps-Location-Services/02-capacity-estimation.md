# Google Maps / Location Services - Capacity Estimation

## Overview

This document calculates the infrastructure requirements for a location services system handling 100 million daily active users with 10 requests per user per day.

---

## Traffic Estimation

### Request Rate

**Given:**
- Daily active users: 100 million
- Requests per user per day: 10
- Operating 24/7

**Calculations:**

```
Total requests per day = 100M × 10
                      = 1 billion requests/day

Requests per hour = 1B / 24 hours
                 = 41.7 million requests/hour

Requests per second (average) = 41.7M / 3,600 seconds
                              = 11,574 requests/second
                              ≈ 11,500 QPS

Peak traffic (4x average during rush hours):
Peak QPS = 11,500 × 4 = 46,000 QPS
```

**Math Verification:**
- Assumptions: 1B requests/day, 24 hours/day, 3,600 seconds/hour, 4x peak multiplier
- Requests/hour: 1,000,000,000 / 24 = 41,666,666.67 requests/hour
- Average QPS: 41,666,666.67 / 3,600 = 11,574.07 QPS ≈ 11,500 QPS (rounded)
- Peak QPS: 11,500 × 4 = 46,000 QPS
- **DOC MATCHES:** Request rate calculations verified ✅

### Request Type Distribution

**Breakdown by operation:**

| Operation Type | % of Requests | QPS (Average) | QPS (Peak) |
|----------------|---------------|---------------|------------|
| Location Search | 40% | 4,600 | 18,400 |
| Route Calculation | 30% | 3,450 | 13,800 |
| Map Tile Requests | 20% | 2,300 | 9,200 |
| ETA Calculation | 5% | 575 | 2,300 |
| Geocoding | 3% | 345 | 1,380 |
| Other | 2% | 230 | 920 |
| **Total** | **100%** | **11,500** | **46,000** |

---

## Storage Estimation

### Map Tiles Storage

**Tile Calculation:**
```
World map coverage:
- Zoom level 0: 1 tile (entire world)
- Zoom level 1: 4 tiles
- Zoom level 2: 16 tiles
- ...
- Zoom level 18: 4^18 = 68.7 billion tiles

Practical coverage:
- Zoom levels 0-10: Global coverage (1.4M tiles)
- Zoom levels 11-15: Urban areas (100M tiles)
- Zoom levels 16-18: Street level (1B tiles)

Total unique tiles: ~1.1 billion tiles
```

**Storage per Tile:**
```
Average tile size: 20 KB (compressed PNG/WebP)
Total storage = 1.1B × 20 KB
             = 22 TB

With 3x replication (CDN):
Total storage = 22 TB × 3 = 66 TB
```

**Math Verification:**
- Assumptions: 1.1B unique tiles, 20 KB per tile, 3x replication
- Base storage: 1,100,000,000 × 20 KB = 22,000,000,000 KB = 22 TB
- With replication: 22 TB × 3 = 66 TB
- **DOC MATCHES:** Storage calculations verified ✅

### Points of Interest (POI) Storage

**POI Data:**
```
Total POIs globally: 100 million
Data per POI:
  - ID: 8 bytes
  - Name: 100 bytes (average)
  - Category: 20 bytes
  - Coordinates (lat, lng): 16 bytes
  - Address: 200 bytes
  - Metadata (rating, hours, etc.): 500 bytes
  Total: ~850 bytes per POI

Total POI storage = 100M × 850 bytes
                 = 85 GB

With indexes (spatial index, text search):
Index overhead: 2x = 170 GB
Total POI storage = 255 GB
```

**Math Verification:**
- Assumptions: 100M POIs, 850 bytes per POI, 2x index overhead
- Base storage: 100,000,000 × 850 bytes = 85,000,000,000 bytes = 85 GB
- With indexes: 85 GB × 2 = 170 GB (index overhead)
- Total: 85 GB + 170 GB = 255 GB
- **DOC MATCHES:** Storage calculations verified ✅

### Traffic Data Storage

**Real-time Traffic:**
```
Road segments globally: 50 million
Updates per segment: 1 per minute
Data per update:
  - Segment ID: 8 bytes
  - Speed: 2 bytes
  - Congestion level: 1 byte
  - Timestamp: 8 bytes
  Total: 19 bytes per update

Updates per second = 50M / 60
                   = 833,333 updates/second

Storage per day = 833,333 × 86,400 × 19 bytes
                = 1.37 TB/day

Retention: 30 days
Total traffic storage = 1.37 TB × 30 = 41 TB
```

### Route Cache Storage

**Cached Routes:**
```
Popular routes: 10 million
Data per route:
  - Route ID: 8 bytes
  - Waypoints: 200 bytes
  - Route geometry: 2 KB (compressed)
  - Metadata: 100 bytes
  Total: ~2.3 KB per route

Total route cache = 10M × 2.3 KB
                  = 23 GB
```

### User Location History

**Location Tracking:**
```
Active users: 100M
Location updates per user: 1 per minute (during navigation)
Average session: 20 minutes
Users navigating per day: 10M (10% of active users)

Location points per day = 10M × 20 minutes × 1 update/min
                        = 200M points/day

Data per point:
  - User ID: 8 bytes
  - Timestamp: 8 bytes
  - Coordinates: 16 bytes
  - Accuracy: 4 bytes
  Total: 36 bytes per point

Storage per day = 200M × 36 bytes
                = 7.2 GB/day

Retention: 90 days
Total location history = 7.2 GB × 90 = 648 GB
```

### Total Storage Summary

| Component           | Size       | Notes                           |
| ------------------- | ---------- | ------------------------------- |
| Map tiles           | 66 TB      | With 3x replication             |
| POI data            | 255 GB     | With indexes                    |
| Traffic data (30d)  | 41 TB      | Real-time + historical          |
| Route cache         | 23 GB      | Popular routes                  |
| Location history    | 648 GB     | 90-day retention                |
| **Total**           | **~108 TB**| Primary storage                 |

---

## Bandwidth Estimation

### Map Tile Bandwidth

**Tile Requests:**
```
Tile requests: 2,300 QPS (average)
Average tile size: 20 KB
Peak multiplier: 4x

Average bandwidth = 2,300 × 20 KB/s
                  = 46 MB/s
                  = 368 Mbps

Peak bandwidth = 368 × 4 = 1,472 Mbps = 1.5 Gbps
```

### API Response Bandwidth

**API Responses:**
```
API requests: 9,200 QPS (non-tile requests)
Average response size: 5 KB
Peak multiplier: 4x

Average bandwidth = 9,200 × 5 KB/s
                  = 46 MB/s
                  = 368 Mbps

Peak bandwidth = 368 × 4 = 1,472 Mbps = 1.5 Gbps
```

### Traffic Data Updates

**Real-time Traffic:**
```
Traffic updates: 833,333 updates/second
Data per update: 19 bytes

Bandwidth = 833,333 × 19 bytes/s
          = 15.8 MB/s
          = 126 Mbps
```

### Total Bandwidth

| Direction       | Bandwidth | Notes                    |
| --------------- | --------- | ------------------------ |
| Outbound (avg)  | 736 Mbps  | Map tiles + API responses|
| Outbound (peak) | 3 Gbps    | 4x average              |
| Traffic updates | 126 Mbps  | Real-time data ingestion|
| **Total**       | **~3.2 Gbps** | Peak requirement         |

---

## Memory Estimation

### Cache Requirements

**Map Tile Cache (CDN):**
```
Hot tiles (frequently accessed): 10M tiles
Tile size: 20 KB
Cache size = 10M × 20 KB = 200 GB
```

**Search Result Cache (Redis):**
```
Popular searches: 1M queries
Average result size: 5 KB
Cache size = 1M × 5 KB = 5 GB
```

**Route Cache (Redis):**
```
Cached routes: 10M routes
Route data: 2.3 KB per route
Cache size = 10M × 2.3 KB = 23 GB
```

**Geospatial Index (Memory):**
```
POI index (QuadTree/R-tree): 100M POIs
Memory per POI: 100 bytes (index overhead)
Index size = 100M × 100 bytes = 10 GB
```

### Total Memory Requirements

| Component              | Memory    | Notes                      |
| ---------------------- | --------- | -------------------------- |
| Map tile cache (CDN)   | 200 GB    | Distributed across CDN     |
| Search cache (Redis)   | 5 GB      | Per Redis instance         |
| Route cache (Redis)    | 23 GB     | Per Redis instance         |
| Geospatial index       | 10 GB     | Per search service instance|
| **Total per region**   | **~38 GB**| In-memory caches           |

---

## Server Estimation

### Search Service

**Calculation:**
```
Search QPS: 4,600 (average)
Queries per server: 1,000/second (limited by geospatial queries)

Servers needed = 4,600 / 1,000 = 5 servers
With redundancy: 8 servers
```

**Search Server Specs:**
```
CPU: 16 cores (geospatial queries are CPU-intensive)
Memory: 32 GB (geospatial index in memory)
Disk: 500 GB SSD
Network: 10 Gbps
```

### Routing Service

**Calculation:**
```
Route QPS: 3,450 (average)
Routes per server: 500/second (routing is computationally expensive)

Servers needed = 3,450 / 500 = 7 servers
With redundancy: 10 servers
```

**Routing Server Specs:**
```
CPU: 32 cores (routing algorithms are CPU-intensive)
Memory: 64 GB (road network graph in memory)
Disk: 1 TB SSD
Network: 10 Gbps
```

### Map Tile Service

**Calculation:**
```
Tile requests: 2,300 QPS (average)
Tiles per server: 2,000/second (mostly cached)

Servers needed = 2,300 / 2,000 = 2 servers
With redundancy: 4 servers
```

**Tile Server Specs:**
```
CPU: 8 cores
Memory: 16 GB
Disk: 2 TB SSD (tile cache)
Network: 10 Gbps
```

### Traffic Service

**Calculation:**
```
Traffic updates: 833,333/second
Updates per server: 100,000/second

Servers needed = 833,333 / 100,000 = 9 servers
With redundancy: 12 servers
```

**Traffic Server Specs:**
```
CPU: 16 cores (streaming processing)
Memory: 32 GB
Disk: 500 GB SSD
Network: 10 Gbps
```

### Server Summary

| Component           | Servers | Specs                           |
| ------------------- | ------- | ------------------------------- |
| Search service      | 8       | 16 cores, 32 GB RAM, 500 GB SSD |
| Routing service     | 10      | 32 cores, 64 GB RAM, 1 TB SSD  |
| Map tile service    | 4       | 8 cores, 16 GB RAM, 2 TB SSD    |
| Traffic service     | 12      | 16 cores, 32 GB RAM, 500 GB SSD|
| Redis cluster       | 6       | 8 cores, 64 GB RAM, 1 TB SSD    |
| PostgreSQL (RDS)    | 3       | 16 cores, 128 GB RAM, 10 TB    |
| **Total**           | **43**  | Plus CDN (managed service)       |

---

## Cost Estimation

### Compute Costs (AWS)

| Component           | Instance Type | Count | Monthly Cost |
| ------------------- | ------------- | ----- | ------------ |
| Search service      | c6i.4xlarge   | 8     | $2,400       |
| Routing service     | c6i.8xlarge   | 10    | $6,000       |
| Map tile service    | c6i.2xlarge   | 4     | $960         |
| Traffic service     | c6i.4xlarge   | 12    | $3,600       |
| Redis cluster       | r6i.2xlarge   | 6     | $2,160       |
| PostgreSQL (RDS)    | db.r6i.4xlarge| 3     | $4,500       |
| **Compute Total**   |               |       | **$19,620**  |

### Storage Costs

| Type                | Size/Requests | Monthly Cost |
| ------------------- | ------------- | ------------ |
| S3 Standard         | 108 TB        | $2,500       |
| S3 Requests         | 1B requests   | $5           |
| EBS (SSD)           | 50 TB         | $5,000       |
| **Storage Total**   |               | **$7,505**   |

### Network Costs

```
Data transfer out: 3 Gbps peak = 9.7 TB/month
CDN costs: $0.085/GB = $825/month
Cross-AZ: ~$500/month

Network total: ~$1,325/month
```

### Third-Party Map Data Costs

```
Google Maps API:
- Geocoding: $5 per 1000 requests
- Directions: $5 per 1000 requests
- Places API: $17 per 1000 requests

Estimated usage:
- Geocoding: 345 QPS × 86,400 = 30M/day = 900M/month
- Directions: 3,450 QPS × 86,400 = 298M/day = 8.9B/month
- Places: 4,600 QPS × 86,400 = 397M/day = 11.9B/month

Costs:
- Geocoding: 900M / 1000 × $5 = $4.5M/month
- Directions: 8.9B / 1000 × $5 = $44.5M/month
- Places: 11.9B / 1000 × $17 = $202.3M/month

Total map API costs: $251.3M/month
```

**Note:** This is extremely expensive. In practice, companies either:
1. Use their own map data (Google, Apple, HERE)
2. Negotiate enterprise pricing (90%+ discount)
3. Build custom solutions for high-volume operations

### Total Monthly Cost

| Category    | Cost         |
| -----------| ------------ |
| Compute     | $19,620     |
| Storage     | $7,505      |
| Network     | $1,325      |
| Map APIs    | $251,300,000|
| Misc (20%)  | $50,270,000 |
| **Total**   | **~$301.6M/month** |

**With enterprise pricing (90% discount on APIs):**
- Map APIs: $25.1M/month
- Total: ~$75.4M/month

### Cost per Request

```
Monthly requests: 1B × 30 = 30 billion requests/month
Monthly cost: $75.4M (with enterprise pricing)

Cost per request = $75.4M / 30B = $0.0025
                 = $2.50 per 1000 requests
```

---

## Scaling Projections

### 10x Scale (1B daily active users)

| Metric          | Current     | 10x Scale   |
| --------------- | ----------- | ----------- |
| Daily users     | 100M        | 1B          |
| QPS (average)   | 11,500      | 115,000     |
| QPS (peak)      | 46,000      | 460,000     |
| Servers         | 43          | 430         |
| Storage         | 108 TB      | 1.08 PB     |
| Monthly cost    | $75M        | $750M       |

### Bottlenecks at Scale

1. **Geospatial queries**: CPU-intensive, hard to parallelize
   - Solution: Shard by geographic region, use distributed indexes

2. **Route calculation**: Graph algorithms are sequential
   - Solution: Pre-compute routes, use contraction hierarchies

3. **Map tile generation**: Expensive rendering
   - Solution: Pre-render tiles, aggressive CDN caching

4. **Traffic data processing**: Millions of updates/second
   - Solution: Stream processing (Kafka + Flink), distributed aggregation

---

## Quick Reference Numbers

### Interview Mental Math

```
100 million daily active users
10 requests per user per day
1 billion requests per day
11,500 QPS average
46,000 QPS peak
108 TB storage
$75M/month (with enterprise pricing)
$2.50 per 1000 requests
```

### Key Ratios

```
Requests per user: 10/day
Search: 40% of requests
Route calculation: 30% of requests
Map tiles: 20% of requests
Cache hit rate: 80% (map tiles)
Route cache hit rate: 60%
```

### Time Estimates

```
Search query: < 200ms
Route calculation: < 500ms
Map tile load: < 100ms
Traffic update: < 30s latency
```

---

## Summary

| Aspect              | Value                                  |
| ------------------- | -------------------------------------- |
| Daily active users  | 100 million                            |
| QPS (average)      | 11,500                                 |
| QPS (peak)          | 46,000                                 |
| Storage             | 108 TB                                 |
| Bandwidth (peak)    | 3.2 Gbps                               |
| Servers             | ~43                                    |
| Monthly cost        | ~$75M (with enterprise pricing)       |
| Cost per request    | $0.0025                                |


