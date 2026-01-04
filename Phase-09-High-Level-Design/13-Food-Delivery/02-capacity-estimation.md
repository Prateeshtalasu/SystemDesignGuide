# Food Delivery - Capacity Estimation (Back-of-Envelope)

## Why Capacity Estimation Matters

A food delivery system has unique capacity challenges:
1. **Peak hour spikes**: Lunch and dinner create 5-10x normal traffic
2. **Three-sided marketplace**: Customers, restaurants, delivery partners
3. **Real-time tracking**: Continuous location updates during delivery
4. **Search-heavy**: Users browse many restaurants before ordering

---

## Traffic Estimation

### Given Assumptions

| Metric | Value | Source |
|--------|-------|--------|
| Daily orders | 500,000 | Business requirement |
| Active restaurants | 200,000 | Across 100 cities |
| Active delivery partners | 100,000 | 30,000 online at peak |
| Daily active users | 2 million | Business requirement |
| Average order value | $25 | Industry average |

### Mental Math Shortcuts

```
1 day = 86,400 seconds ≈ 100,000 seconds
1 million requests/day = ~12 QPS
Peak is typically 5x average for food delivery (meal times)
```

### Order QPS

**Order Placement:**
```
500,000 orders/day
= 500K / 86,400 seconds
≈ 5.8 orders/second (average)

Peak (dinner rush, 5x) = 29 orders/second
Super peak (Friday dinner) = 50 orders/second
```

**Math Verification:**
- Assumptions: 500K orders/day, 86,400 seconds/day, 5x peak multiplier
- Average QPS: 500,000 / 86,400 = 5.787 orders/sec ≈ 5.8 orders/sec (rounded)
- Peak QPS: 5.8 × 5 = 29 orders/sec
- Super peak: ~8.6x average = 5.8 × 8.6 ≈ 50 orders/sec
- **DOC MATCHES:** Order QPS calculations verified ✅

### Search & Browse QPS

Users browse extensively before ordering:
```
Average searches per order = 10
Daily searches = 500K × 10 = 5 million

Search QPS (average) = 5M / 86,400 ≈ 58/second
Search QPS (peak, 5x) = 290/second
```

**Math Verification:**
- Assumptions: 5M searches/day, 86,400 seconds/day, 5x peak multiplier
- Average QPS: 5,000,000 / 86,400 = 57.87 QPS ≈ 58 QPS (rounded)
- Peak QPS: 58 × 5 = 290 QPS
- **DOC MATCHES:** Search QPS verified ✅

Menu views per order = 5
Daily menu views = 500K × 5 = 2.5 million
Menu view QPS (average) = 29/second
Menu view QPS (peak) = 145/second
```

### Location Update QPS

**Delivery Partner Location Updates:**
```
Active delivery partners at peak = 30,000
Partners on active delivery = 15,000 (50%)
Updates per partner per second = 0.25 (every 4 seconds)

Location updates/second = 30,000 × 0.25 = 7,500/second
```

**Math Verification:**
- Assumptions: 30,000 active delivery partners at peak, 0.25 updates/sec per partner (1 update per 4 seconds)
- Calculation: 30,000 × 0.25 = 7,500 updates/sec
- **DOC MATCHES:** Location update QPS verified ✅

**Customer Tracking Requests:**
```
Active orders being tracked = 15,000
Tracking requests per order = 0.2/second (every 5 seconds)

Tracking QPS = 15,000 × 0.2 = 3,000/second
```

**Math Verification:**
- Assumptions: 15,000 active orders being tracked, 0.2 requests/sec per order (1 request per 5 seconds)
- Calculation: 15,000 × 0.2 = 3,000 QPS
- **DOC MATCHES:** Tracking request QPS verified ✅

### API Requests Summary

| Operation | Average QPS | Peak QPS (5x) |
|-----------|-------------|---------------|
| Restaurant search | 58 | 290 |
| Menu views | 29 | 145 |
| Order placement | 6 | 30 |
| Order status updates | 20 | 100 |
| Location updates (partners) | 7,500 | 10,000 |
| Tracking requests | 3,000 | 5,000 |
| Payment processing | 6 | 30 |
| **Total** | **~10,600** | **~15,600** |

### Geographic Distribution

Assuming US-focused service:
- Major metros (NYC, LA, SF, Chicago): 50%
- Secondary cities: 35%
- Suburban areas: 15%

Peak hours by timezone:
- Lunch: 11am-2pm local
- Dinner: 5pm-9pm local

---

## Storage Estimation

### Data Model Sizes

**Restaurant Record:**

| Field | Type | Size |
|-------|------|------|
| restaurant_id | UUID | 16 bytes |
| name | VARCHAR(100) | 50 bytes |
| description | TEXT | 500 bytes |
| address | VARCHAR(255) | 150 bytes |
| location | POINT | 16 bytes |
| cuisine_types | VARCHAR[] | 100 bytes |
| rating | DECIMAL | 4 bytes |
| operating_hours | JSON | 200 bytes |
| delivery_radius | INT | 4 bytes |
| photos | JSON | 500 bytes |
| settings | JSON | 300 bytes |

**Total per restaurant: ~1.8 KB → round to 2 KB**

**Menu Item Record:**

| Field | Type | Size |
|-------|------|------|
| item_id | UUID | 16 bytes |
| restaurant_id | UUID | 16 bytes |
| name | VARCHAR(100) | 50 bytes |
| description | TEXT | 300 bytes |
| price | DECIMAL | 8 bytes |
| category | VARCHAR(50) | 30 bytes |
| photo_url | VARCHAR(255) | 100 bytes |
| customizations | JSON | 500 bytes |
| availability | BOOLEAN | 1 byte |
| prep_time | INT | 4 bytes |

**Total per menu item: ~1 KB**

**Order Record:**

| Field | Type | Size |
|-------|------|------|
| order_id | UUID | 16 bytes |
| customer_id | UUID | 16 bytes |
| restaurant_id | UUID | 16 bytes |
| delivery_partner_id | UUID | 16 bytes |
| items | JSON | 1000 bytes |
| subtotal | DECIMAL | 8 bytes |
| tax | DECIMAL | 8 bytes |
| delivery_fee | DECIMAL | 8 bytes |
| tip | DECIMAL | 8 bytes |
| total | DECIMAL | 8 bytes |
| status | ENUM | 1 byte |
| delivery_address | JSON | 200 bytes |
| special_instructions | TEXT | 200 bytes |
| timestamps | JSON | 200 bytes |

**Total per order: ~1.7 KB → round to 2 KB**

**Delivery Partner Location:**

| Field | Type | Size |
|-------|------|------|
| partner_id | UUID | 16 bytes |
| latitude | DOUBLE | 8 bytes |
| longitude | DOUBLE | 8 bytes |
| timestamp | TIMESTAMP | 8 bytes |
| heading | FLOAT | 4 bytes |
| speed | FLOAT | 4 bytes |

**Total per location point: ~50 bytes**

### Storage Calculations

**Restaurant & Menu Storage:**
```
Restaurants: 200,000 × 2 KB = 400 MB
Menu items: 20 million × 1 KB = 20 GB
Total catalog: ~21 GB
```

**Order Storage:**
```
Orders per day: 500,000
Order record size: 2 KB
Daily order storage: 500K × 2 KB = 1 GB

Per year: 1 GB × 365 = 365 GB
5-year retention: 1.8 TB
```

**Location History Storage:**
```
Active deliveries per day: 500,000
Average delivery time: 30 minutes
Location updates per delivery: 30 × 60 / 4 = 450 points
Storage per delivery: 450 × 50 bytes = 22.5 KB

Daily: 500K × 22.5 KB = 11 GB
Per year: 11 GB × 365 = 4 TB
```

**Search Index Storage:**
```
Restaurant index (Elasticsearch): 200K × 5 KB = 1 GB
Menu item index: 20M × 500 bytes = 10 GB
Total search index: ~11 GB
```

### Storage Summary

| Data Type | 1 Year | 5 Years | Notes |
|-----------|--------|---------|-------|
| Restaurant/Menu data | 21 GB | 30 GB | Slow growth |
| Order records | 365 GB | 1.8 TB | Linear growth |
| Location history | 4 TB | 20 TB | Can be aggregated |
| Search indexes | 11 GB | 15 GB | Rebuilt periodically |
| User data | 10 GB | 20 GB | 2M users × 5 KB |
| **Total** | ~4.5 TB | ~22 TB | Before replication |

**With Replication (3x):**
```
5-year storage with replication: 22 TB × 3 = 66 TB
```

---

## Bandwidth Estimation

### Incoming Bandwidth

**Search Requests:**
```
Request size: 500 bytes
Peak QPS: 290
Incoming: 290 × 500 = 145 KB/s ≈ 1.2 Mbps
```

**Order Requests:**
```
Request size: 2 KB (includes items)
Peak QPS: 30
Incoming: 30 × 2 KB = 60 KB/s ≈ 0.5 Mbps
```

**Location Updates:**
```
Update size: 100 bytes
Peak QPS: 10,000
Incoming: 10,000 × 100 = 1 MB/s ≈ 8 Mbps
```

**Total Incoming: ~15 Mbps peak**

### Outgoing Bandwidth

**Search Results:**
```
Response size: 10 KB (list of restaurants)
Peak QPS: 290
Outgoing: 290 × 10 KB = 2.9 MB/s ≈ 23 Mbps
```

**Menu Data:**
```
Response size: 50 KB (full menu)
Peak QPS: 145
Outgoing: 145 × 50 KB = 7.25 MB/s ≈ 58 Mbps
```

**Images (via CDN):**
```
Image requests: 1,000/second
Average image: 100 KB
CDN bandwidth: 100 MB/s ≈ 800 Mbps
```

### Bandwidth Summary

| Direction | Average | Peak |
|-----------|---------|------|
| Incoming | 5 Mbps | 15 Mbps |
| Outgoing (API) | 30 Mbps | 100 Mbps |
| CDN (images) | 400 Mbps | 1 Gbps |

---

## Memory Estimation (Cache & Real-time Data)

### Restaurant Cache

Popular restaurants cached for fast search:
```
Hot restaurants (20%): 40,000
Restaurant data: 5 KB (includes basic menu)
Memory: 40,000 × 5 KB = 200 MB
```

### Menu Cache

Frequently accessed menus:
```
Hot menus (10%): 20,000 restaurants
Menu size: 50 KB average
Memory: 20,000 × 50 KB = 1 GB
```

### Active Order Cache

All active orders in memory:
```
Active orders at peak: 50,000
Order state: 500 bytes
Memory: 50,000 × 500 = 25 MB
```

### Delivery Partner Location Cache

All online partners:
```
Online partners: 30,000
Location data: 100 bytes
Memory: 30,000 × 100 = 3 MB
```

### Session Cache

User sessions:
```
Concurrent users: 200,000
Session data: 500 bytes
Memory: 200,000 × 500 = 100 MB
```

### Cache Summary

| Cache Type | Size | TTL |
|------------|------|-----|
| Restaurant data | 200 MB | 1 hour |
| Menu data | 1 GB | 15 minutes |
| Active orders | 25 MB | Order duration |
| Partner locations | 3 MB | 30 seconds |
| User sessions | 100 MB | 30 minutes |
| Search results | 500 MB | 5 minutes |
| **Total** | **~2 GB** | |

**Recommendation:** 4 GB Redis cluster for headroom.

---

## Server Estimation

### API Gateway / Load Balancer

```
Peak requests: 15,600/second
Each server handles: 5,000/second
Servers needed: 4
With redundancy: 6 servers
```

### Search Service

```
Peak search QPS: 290
Each Elasticsearch node: 100 QPS (with aggregations)
Nodes needed: 3
With redundancy: 6 nodes (3 primary + 3 replica)
```

### Order Service

```
Peak order operations: 200/second
Each server handles: 100/second
Servers needed: 2
With redundancy: 4 servers
```

### Location Service

```
Peak location updates: 10,000/second
Each server handles: 3,000/second
Servers needed: 4
With redundancy: 6 servers
```

### Restaurant Service

```
Peak menu requests: 145/second
Each server handles: 100/second
Servers needed: 2
With redundancy: 4 servers
```

### Database Servers

**PostgreSQL:**
- 1 Primary (writes)
- 3 Read replicas
- Total: 4 servers

**Redis:**
- 3-node cluster
- Total: 3 servers

**Elasticsearch:**
- 6 nodes (3 primary + 3 replica)

### Server Summary

| Component | Count | Specs |
|-----------|-------|-------|
| Load Balancer | 2 | HA pair |
| API Gateway | 6 | 4 CPU, 8 GB RAM |
| Search Service | 6 | 8 CPU, 32 GB RAM |
| Order Service | 4 | 4 CPU, 8 GB RAM |
| Location Service | 6 | 4 CPU, 8 GB RAM |
| Restaurant Service | 4 | 4 CPU, 8 GB RAM |
| Notification Service | 4 | 4 CPU, 8 GB RAM |
| PostgreSQL | 4 | 16 CPU, 64 GB RAM |
| Redis Cluster | 3 | 8 GB RAM each |
| Elasticsearch | 6 | 8 CPU, 32 GB RAM |
| Kafka | 6 | 8 CPU, 32 GB RAM |
| **Total** | **51** | |

---

## Peak Hour Analysis

### Traffic Pattern

```
Hour     | Orders/Hour | Multiplier
---------|-------------|------------
6am-10am |    10,000   |    0.5x
10am-12pm|    30,000   |    1.5x
12pm-2pm |    60,000   |    3.0x (Lunch peak)
2pm-5pm  |    15,000   |    0.75x
5pm-7pm  |    50,000   |    2.5x
7pm-9pm  |    80,000   |    4.0x (Dinner peak)
9pm-11pm |    30,000   |    1.5x
11pm-6am |     5,000   |    0.25x
```

### Peak Hour Capacity Planning

During dinner peak (7pm-9pm):
```
Orders: 80,000 in 2 hours = 40,000/hour = 11 orders/second
Searches: 11 × 10 = 110/second
Active orders: 40,000 × 0.75 hours = 30,000 concurrent
```

System must handle 4x average load during peak.

---

## Common FAANG Interview Numbers

| Metric | Value |
|--------|-------|
| Average order value | $25 |
| Platform commission | 20-30% |
| Delivery time target | 30-45 minutes |
| Restaurant prep time | 15-20 minutes |
| Delivery distance | 3-5 km average |
| Orders per delivery partner/hour | 2-3 |
| Partner location update frequency | 4-10 seconds |

---

## Growth Projections

### Year-over-Year Growth

Assuming 40% YoY growth:

| Year | Orders/Day | Restaurants | Storage |
|------|------------|-------------|---------|
| Year 1 | 500K | 200K | 4.5 TB |
| Year 2 | 700K | 280K | 7 TB |
| Year 3 | 980K | 400K | 11 TB |
| Year 4 | 1.4M | 560K | 16 TB |
| Year 5 | 2M | 800K | 24 TB |

### Scaling Triggers

| Metric | Threshold | Action |
|--------|-----------|--------|
| Search latency P99 | > 500ms | Add Elasticsearch nodes |
| Order service CPU | > 70% | Scale horizontally |
| Database connections | > 80% | Add read replicas |
| Cache hit rate | < 90% | Increase cache size |
| Delivery assignment time | > 30s | Optimize matching |

---

## Cost Estimation (AWS Pricing)

### Monthly Costs (Rough Estimate)

| Component | Specs | Monthly Cost |
|-----------|-------|--------------|
| EC2 (App Servers) | 30 × c5.xlarge | $4,500 |
| EC2 (Database) | 4 × r5.4xlarge | $4,000 |
| ElastiCache (Redis) | 3 × r5.large | $500 |
| Elasticsearch | 6 × r5.2xlarge | $4,000 |
| RDS Storage | 10 TB SSD | $1,000 |
| Kafka (MSK) | 6 × kafka.m5.large | $2,000 |
| S3 (Images) | 5 TB | $100 |
| CloudFront (CDN) | 10 TB transfer | $1,000 |
| Maps API | 50M requests | $10,000 |
| Load Balancer | 2 × ALB | $500 |
| **Total** | | **~$27,600/month** |

### Cost per Order

```
Total monthly cost = $27,600
Total monthly orders = 500K × 30 = 15M orders

Cost per order = $27,600 / 15M = $0.00184
≈ $0.002 per order

Revenue per order (25% commission on $25):
= $25 × 0.25 = $6.25

Infrastructure cost as % of revenue:
= $0.002 / $6.25 = 0.03%
```

Infrastructure is negligible compared to delivery partner payments.

---

## Summary Table

| Metric | Value |
|--------|-------|
| **Traffic** | |
| Orders/second (peak) | 30 |
| Searches/second (peak) | 290 |
| Location updates/second | 10,000 |
| **Storage** | |
| Catalog data | 21 GB |
| Order data (5 years) | 1.8 TB |
| Location history (5 years) | 20 TB |
| **Memory** | |
| Total cache size | 2 GB |
| Target cache hit rate | 90%+ |
| **Bandwidth** | |
| Peak API bandwidth | 100 Mbps |
| Peak CDN bandwidth | 1 Gbps |
| **Infrastructure** | |
| Application servers | ~30 |
| Database servers | ~4 |
| Search nodes | ~6 |
| **Cost** | |
| Monthly infrastructure | ~$27,600 |
| Cost per order | ~$0.002 |

---

## Interview Tips

### Show Your Work

Always show calculations step by step:
- State assumptions clearly
- Account for peak hours (5x for food delivery)
- Consider all three sides of marketplace

### Key Insight for Food Delivery

The unique challenge is **peak hour handling**:
- Lunch and dinner create massive spikes
- System must scale 5x for 4 hours daily
- Pre-scaling based on time-of-day patterns

### Common Mistakes

1. **Ignoring peak hours**: Average QPS is misleading
2. **Underestimating search load**: Users browse a lot before ordering
3. **Forgetting images**: Menu images dominate bandwidth
4. **Not considering all actors**: Restaurants and delivery partners have traffic too

