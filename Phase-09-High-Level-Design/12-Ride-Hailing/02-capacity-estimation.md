# Ride Hailing - Capacity Estimation (Back-of-Envelope)

## Why Capacity Estimation Matters

A ride-hailing system has unique capacity challenges:
1. **Real-time location tracking**: Millions of location updates per second
2. **Geospatial queries**: Finding nearby drivers efficiently
3. **Event-driven architecture**: High-frequency state changes
4. **Peak handling**: Rush hours create 5-10x traffic spikes

---

## Traffic Estimation

### Given Assumptions

| Metric | Value | Source |
|--------|-------|--------|
| Daily Active Users (Riders) | 10 million | Business requirement |
| Active Drivers | 1 million | 10:1 rider to driver ratio |
| Rides per day | 5 million | 50% of DAU take rides |
| Peak concurrent users | 500,000 | 5% of DAU at peak |
| Peak concurrent drivers | 200,000 | 20% of drivers online at peak |

### Mental Math Shortcuts

```
1 day = 86,400 seconds ≈ 100,000 seconds (for easy math)
1 million requests/day = ~12 QPS
1 billion requests/day = ~12,000 QPS
Peak is typically 3-5x average
```

### Location Update QPS

**Driver Location Updates:**

Drivers send location updates every 4 seconds while online.

```
Active drivers at peak = 200,000
Updates per driver per second = 1/4 = 0.25

Location updates/second (average) = 200,000 × 0.25 = 50,000 updates/sec

Peak (rush hour, 2x) = 100,000 updates/sec
```

**Math Verification:**
- Assumptions: 200,000 active drivers at peak, 0.25 updates/sec per driver (1 update per 4 seconds), 2x peak multiplier
- Average: 200,000 × 0.25 = 50,000 updates/sec
- Peak: 50,000 × 2 = 100,000 updates/sec
- **DOC MATCHES:** Location update QPS verified ✅

**Rider Location Updates:**

Riders update location during active rides (pickup wait + trip).

```
Active rides at any moment = 100,000 (estimate)
Updates per rider per second = 0.1 (every 10 seconds)

Rider location updates = 100,000 × 0.1 = 10,000 updates/sec
```

**Total Location Updates:**
```
Average: 60,000 updates/sec
Peak: 110,000 updates/sec
```

### Ride Request QPS

**Ride Requests:**
```
5 million rides/day
= 5M / 86,400 seconds
≈ 58 rides/second (average)

Peak (rush hour, 5x) = 290 rides/second
```

**Math Verification:**
- Assumptions: 5M rides/day, 86,400 seconds/day, 5x peak multiplier
- Average QPS: 5,000,000 / 86,400 = 57.87 rides/sec ≈ 58 rides/sec (rounded)
- Peak QPS: 58 × 5 = 290 rides/sec
- **DOC MATCHES:** Ride request QPS verified ✅

### Matching Queries QPS

Each ride request triggers multiple matching queries:
- Initial nearby driver search
- Driver notification (fan-out to multiple drivers)
- Re-matching if first driver declines

```
Matching queries per ride = 5 (average)
Matching queries/second (average) = 58 × 5 = 290 QPS
Matching queries/second (peak) = 290 × 5 = 1,450 QPS
```

**Math Verification:**
- Assumptions: 58 rides/sec average, 5 matching queries per ride, 5x peak multiplier
- Average: 58 × 5 = 290 QPS
- Peak: 290 × 5 = 1,450 QPS
- **DOC MATCHES:** Matching query QPS verified ✅

### API Requests QPS

| Operation | Average QPS | Peak QPS (5x) |
|-----------|-------------|---------------|
| Location updates (drivers) | 50,000 | 100,000 |
| Location updates (riders) | 10,000 | 20,000 |
| Ride requests | 58 | 290 |
| Matching queries | 290 | 1,450 |
| ETA calculations | 500 | 2,500 |
| Payment processing | 58 | 290 |
| Rating submissions | 116 | 580 |
| **Total** | **~61,000** | **~125,000** |

### Geographic Distribution

Assuming US-focused service:
- West Coast (California): 30%
- East Coast (NY, FL): 35%
- Central (Texas, Chicago): 25%
- Other: 10%

This affects:
- Data center placement
- CDN edge locations
- Database replication strategy

---

## Storage Estimation

### Data Model Sizes

**User Record (Rider/Driver):**

| Field | Type | Size |
|-------|------|------|
| user_id | UUID | 16 bytes |
| phone_number | VARCHAR(15) | 15 bytes |
| email | VARCHAR(100) | 50 bytes avg |
| name | VARCHAR(100) | 30 bytes avg |
| profile_photo_url | VARCHAR(255) | 100 bytes |
| rating | DECIMAL(3,2) | 4 bytes |
| total_rides | INT | 4 bytes |
| created_at | TIMESTAMP | 8 bytes |
| updated_at | TIMESTAMP | 8 bytes |
| payment_methods | JSON | 200 bytes |
| preferences | JSON | 100 bytes |

**Total per user: ~550 bytes → round to 1 KB**

**Driver Additional Data:**

| Field | Type | Size |
|-------|------|------|
| license_number | VARCHAR(20) | 20 bytes |
| vehicle_info | JSON | 300 bytes |
| insurance_info | JSON | 200 bytes |
| documents | JSON | 200 bytes |
| bank_account | JSON | 150 bytes |

**Total per driver: ~1 KB additional → 2 KB total**

**Ride Record:**

| Field | Type | Size |
|-------|------|------|
| ride_id | UUID | 16 bytes |
| rider_id | UUID | 16 bytes |
| driver_id | UUID | 16 bytes |
| pickup_location | POINT | 16 bytes |
| dropoff_location | POINT | 16 bytes |
| pickup_address | VARCHAR(255) | 100 bytes |
| dropoff_address | VARCHAR(255) | 100 bytes |
| status | ENUM | 1 byte |
| vehicle_type | ENUM | 1 byte |
| fare_estimate | DECIMAL | 8 bytes |
| final_fare | DECIMAL | 8 bytes |
| distance_km | DECIMAL | 4 bytes |
| duration_minutes | INT | 4 bytes |
| surge_multiplier | DECIMAL | 4 bytes |
| created_at | TIMESTAMP | 8 bytes |
| started_at | TIMESTAMP | 8 bytes |
| completed_at | TIMESTAMP | 8 bytes |
| route_polyline | TEXT | 500 bytes |
| rider_rating | TINYINT | 1 byte |
| driver_rating | TINYINT | 1 byte |

**Total per ride: ~850 bytes → round to 1 KB**

**Location History (for trip replay):**

| Field | Type | Size |
|-------|------|------|
| ride_id | UUID | 16 bytes |
| timestamp | TIMESTAMP | 8 bytes |
| latitude | DOUBLE | 8 bytes |
| longitude | DOUBLE | 8 bytes |
| speed | FLOAT | 4 bytes |
| heading | FLOAT | 4 bytes |

**Total per location point: ~50 bytes**

### Storage Calculations

**User Storage:**
```
Riders: 10M × 1 KB = 10 GB
Drivers: 1M × 2 KB = 2 GB
Total users: 12 GB
```

**Ride Storage:**
```
Rides per day: 5 million
Ride record size: 1 KB
Daily ride storage: 5M × 1 KB = 5 GB

Per year: 5 GB × 365 = 1.8 TB
5-year retention: 9 TB
```

**Location History Storage:**
```
Average ride duration: 15 minutes
Location updates per ride: 15 × 60 / 4 = 225 points
Storage per ride: 225 × 50 bytes = 11.25 KB

Daily: 5M rides × 11.25 KB = 56 GB
Per year: 56 GB × 365 = 20 TB
```

**Driver Location (Real-time):**
```
Active drivers: 200,000
Location record: 50 bytes
Real-time storage: 200,000 × 50 = 10 MB (in-memory)
```

### Storage Summary

| Data Type | 1 Year | 5 Years | Notes |
|-----------|--------|---------|-------|
| User data | 12 GB | 20 GB | Slow growth |
| Ride records | 1.8 TB | 9 TB | Linear growth |
| Location history | 20 TB | 100 TB | Largest component |
| Driver locations (real-time) | 10 MB | 10 MB | In-memory only |
| **Total** | ~22 TB | ~110 TB | Before replication |

**With Replication (3x):**
```
5-year storage with replication: 110 TB × 3 = 330 TB
```

---

## Bandwidth Estimation

### Incoming Bandwidth

**Location Updates:**
```
Update size: 100 bytes (location + metadata)
Peak updates: 110,000/second
Incoming: 110,000 × 100 bytes = 11 MB/s ≈ 88 Mbps
```

**Ride Requests:**
```
Request size: 500 bytes
Peak requests: 290/second
Incoming: 290 × 500 = 145 KB/s ≈ 1.2 Mbps
```

**Total Incoming: ~100 Mbps peak**

### Outgoing Bandwidth

**Location Broadcasts (to riders tracking drivers):**
```
Active riders tracking: 100,000
Updates per second: 0.25 (every 4 seconds)
Update size: 100 bytes
Outgoing: 100,000 × 0.25 × 100 = 2.5 MB/s ≈ 20 Mbps
```

**Map Data:**
```
Map tile requests: 10,000/second
Tile size: 50 KB average
Outgoing: 10,000 × 50 KB = 500 MB/s ≈ 4 Gbps
```

Note: Map data typically served via CDN, so actual origin bandwidth is much lower.

**Push Notifications:**
```
Notifications: 1,000/second (ride updates, promos)
Size: 1 KB each
Outgoing: 1 MB/s ≈ 8 Mbps
```

### Bandwidth Summary

| Direction | Average | Peak |
|-----------|---------|------|
| Incoming | 50 Mbps | 100 Mbps |
| Outgoing (excluding CDN) | 30 Mbps | 50 Mbps |
| CDN (map tiles) | 2 Gbps | 5 Gbps |

---

## Memory Estimation (Cache & Real-time Data)

### Driver Location Cache

All online driver locations must be in memory for fast geospatial queries.

```
Online drivers (peak): 200,000
Location record: 100 bytes (lat, long, heading, speed, timestamp, driver_id)
Memory: 200,000 × 100 = 20 MB

With geospatial index overhead (2x): 40 MB
```

### Active Ride Cache

All ongoing rides cached for real-time updates.

```
Active rides (peak): 100,000
Ride state: 500 bytes
Memory: 100,000 × 500 = 50 MB
```

### User Session Cache

```
Concurrent users: 500,000
Session data: 200 bytes
Memory: 500,000 × 200 = 100 MB
```

### Surge Pricing Cache

```
Geographic zones: 10,000 (city blocks/areas)
Surge data per zone: 100 bytes
Memory: 10,000 × 100 = 1 MB
```

### Cache Summary

| Cache Type | Size | TTL |
|------------|------|-----|
| Driver locations | 40 MB | 30 seconds |
| Active rides | 50 MB | Duration of ride |
| User sessions | 100 MB | 30 minutes |
| Surge pricing | 1 MB | 5 minutes |
| ETA cache | 50 MB | 1 minute |
| **Total** | **~250 MB** | |

**Recommendation:** 1 GB Redis per region, clustered for HA.

---

## Server Estimation

### Location Service (Handles Location Updates)

```
Peak location updates: 110,000/second
Each server handles: 10,000 updates/second
Servers needed: 11
With 50% headroom: 17 servers
```

### Matching Service (Handles Ride Requests)

```
Peak matching queries: 1,450/second
Each server handles: 500 queries/second (CPU-intensive geospatial)
Servers needed: 3
With 50% headroom: 5 servers
```

### Trip Service (Manages Ride State)

```
Peak state updates: 5,000/second
Each server handles: 2,000 updates/second
Servers needed: 3
With 50% headroom: 5 servers
```

### WebSocket Servers (Real-time Communication)

```
Concurrent connections: 700,000 (drivers + active riders)
Connections per server: 50,000
Servers needed: 14
With 50% headroom: 21 servers
```

### Database Servers

**PostgreSQL (Transactional Data):**
- 1 Primary (writes)
- 3 Read replicas (reads)
- Total: 4 servers

**Redis (Real-time Data):**
- 3-node cluster per region
- 2 regions
- Total: 6 servers

**Time-series DB (Location History):**
- 3-node cluster (TimescaleDB or InfluxDB)

### Server Summary

| Component | Count | Specs |
|-----------|-------|-------|
| Load Balancer | 2 | HA pair |
| Location Service | 17 | 4 CPU, 8 GB RAM |
| Matching Service | 5 | 8 CPU, 16 GB RAM |
| Trip Service | 5 | 4 CPU, 8 GB RAM |
| WebSocket Servers | 21 | 4 CPU, 16 GB RAM |
| API Gateway | 6 | 4 CPU, 8 GB RAM |
| PostgreSQL | 4 | 16 CPU, 64 GB RAM, SSD |
| Redis Cluster | 6 | 8 GB RAM each |
| TimescaleDB | 3 | 8 CPU, 32 GB RAM |
| Kafka Cluster | 6 | 8 CPU, 32 GB RAM |
| **Total** | **75** | |

---

## Common FAANG Interview Numbers

| Metric | Value |
|--------|-------|
| GPS update frequency | 1-4 seconds |
| Acceptable matching time | 10-30 seconds |
| WebSocket connections per server | 50,000-100,000 |
| Geospatial query latency (Redis) | < 5ms |
| PostgreSQL write latency | 5-10ms |
| Kafka message latency | 2-10ms |
| Mobile network latency | 50-200ms |

---

## Growth Projections

### Year-over-Year Growth

Assuming 30% YoY growth:

| Year | DAU | Rides/Day | Location Updates/sec |
|------|-----|-----------|---------------------|
| Year 1 | 10M | 5M | 60K |
| Year 2 | 13M | 6.5M | 78K |
| Year 3 | 17M | 8.5M | 101K |
| Year 4 | 22M | 11M | 132K |
| Year 5 | 29M | 14M | 170K |

### Scaling Triggers

| Metric | Threshold | Action |
|--------|-----------|--------|
| Location service CPU | > 70% | Add more instances |
| Matching latency P99 | > 5 seconds | Scale matching service |
| WebSocket connections | > 80% capacity | Add WS servers |
| Redis memory | > 70% | Add cluster nodes |
| Database connections | > 80% | Add read replicas |

---

## Cost Estimation (AWS Pricing)

### Monthly Costs (Rough Estimate)

| Component | Specs | Monthly Cost |
|-----------|-------|--------------|
| EC2 (App Servers) | 60 × c5.xlarge | $7,500 |
| EC2 (Database) | 4 × r5.4xlarge | $4,000 |
| ElastiCache (Redis) | 6 × r5.large | $1,000 |
| RDS Storage | 10 TB SSD | $1,000 |
| Kafka (MSK) | 6 × kafka.m5.large | $2,000 |
| Data Transfer | 500 TB/month | $25,000 |
| Maps API | 100M requests | $20,000 |
| Load Balancer | 2 × ALB | $500 |
| **Total** | | **~$61,000/month** |

### Cost per Ride

```
Total monthly cost = $61,000
Total monthly rides = 5M × 30 = 150M rides

Cost per ride = $61,000 / 150M = $0.0004
= $0.40 per 1000 rides
```

With 20% commission on average $15 fare:
```
Revenue per ride = $15 × 0.20 = $3
Cost per ride = $0.0004
Gross margin = 99.99%
```

Infrastructure cost is negligible compared to driver payments, marketing, and support.

---

## Summary Table

| Metric | Value |
|--------|-------|
| **Traffic** | |
| Location updates/second (peak) | 110,000 |
| Ride requests/second (peak) | 290 |
| Concurrent WebSocket connections | 700,000 |
| **Storage** | |
| User data | 12 GB |
| Ride data (5 years) | 9 TB |
| Location history (5 years) | 100 TB |
| **Memory** | |
| Real-time cache | 250 MB |
| Driver location index | 40 MB |
| **Bandwidth** | |
| Peak bandwidth (excluding CDN) | 150 Mbps |
| **Infrastructure** | |
| Application servers | ~60 |
| Database servers | ~13 |
| WebSocket servers | ~21 |
| **Cost** | |
| Monthly infrastructure | ~$61,000 |
| Cost per ride | ~$0.0004 |

---

## Interview Tips

### Show Your Work

Always show calculations step by step:
- State assumptions clearly
- Round numbers for easy mental math
- Sanity check your results

### Key Insight for Ride-Hailing

The unique challenge is **real-time geospatial queries at scale**:
- 100K+ location updates per second
- Sub-second geospatial matching
- WebSocket connections for real-time updates

### Common Mistakes

1. **Underestimating location update volume**: Every driver sends updates every few seconds
2. **Ignoring map data costs**: Maps API can be expensive at scale
3. **Forgetting WebSocket scaling**: Stateful connections are harder to scale
4. **Not considering surge pricing data**: Needs real-time demand calculation

