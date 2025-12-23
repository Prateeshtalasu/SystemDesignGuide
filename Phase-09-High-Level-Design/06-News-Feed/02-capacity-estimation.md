# News Feed - Capacity Estimation

## Overview

This document calculates the infrastructure requirements for a news feed system serving 500 million daily active users.

---

## Traffic Estimation

### User Activity

**Given:**
- Daily Active Users (DAU): 500 million
- Monthly Active Users (MAU): 2 billion
- Average follows per user: 200

**Feed Requests:**
```
Feed loads per user per day: 10 (app opens)
Feed refreshes per user per day: 50 (pull to refresh, scroll)
Feed requests per user per day: 60

Total feed requests/day = 500M × 60 = 30 billion
```

**Peak QPS Calculation:**
```
Average QPS = 30B / 86,400 = 347,000 QPS
Peak factor = 3x (evening hours)
Peak QPS = 347,000 × 3 = ~1 million QPS
```

### Post Creation

**Post Volume:**
```
Posts per DAU per day: 0.5 (not everyone posts daily)
Total posts/day = 500M × 0.5 = 250 million posts/day

Posts per second (avg) = 250M / 86,400 = 2,900 posts/sec
Posts per second (peak) = 2,900 × 3 = 8,700 posts/sec
```

### Fan-out Traffic

**For Regular Users (< 10K followers):**
```
Regular users: 99% of posters
Posts from regular users: 250M × 0.99 = 247.5M
Average followers: 200

Fan-out writes = 247.5M × 200 = 49.5 billion writes/day
Fan-out writes/sec = 49.5B / 86,400 = 573,000 writes/sec
```

**For Celebrities (> 10K followers):**
```
Celebrity users: 1% of posters = 2.5M posts/day
These are NOT fanned out on write (handled on read)
```

---

## Storage Estimation

### Post Storage

**Per Post:**
```
- Post ID: 8 bytes
- User ID: 8 bytes
- Content: 500 bytes average (text)
- Media references: 100 bytes (URLs to images/videos)
- Metadata: 100 bytes (timestamps, privacy, etc.)
- Engagement counters: 50 bytes

Total per post: ~750 bytes
```

**Daily Post Storage:**
```
Posts per day: 250 million
Storage per day = 250M × 750 bytes = 187.5 GB/day
```

**Annual Post Storage:**
```
Storage per year = 187.5 GB × 365 = 68.4 TB/year
With 3x replication = 205 TB/year
```

### Feed Cache Storage

**Per User Feed:**
```
Feed entries cached: 500 posts
Each entry: post_id (8 bytes) + score (4 bytes) + timestamp (8 bytes) = 20 bytes
Feed size per user = 500 × 20 = 10 KB
```

**Total Feed Cache:**
```
Active users with cached feeds: 500M (DAU)
Total feed cache = 500M × 10 KB = 5 TB

With hot/cold tiering:
- Hot (last 24h active): 100M users × 10 KB = 1 TB
- Warm (last 7 days): 200M users × 10 KB = 2 TB
- Cold (others): Regenerate on demand
```

### Social Graph Storage

**Follow Relationships:**
```
Total users: 2 billion
Average follows: 200
Total edges: 2B × 200 = 400 billion edges

Per edge: follower_id (8 bytes) + following_id (8 bytes) = 16 bytes
Total graph storage = 400B × 16 = 6.4 TB
With indexes and replication (3x) = ~20 TB
```

### Media Storage (References Only)

```
Posts with images: 30% of 250M = 75M/day
Average images per post: 2
Image storage (actual files in CDN): handled separately
Media metadata: 75M × 2 × 100 bytes = 15 GB/day
```

### Total Storage Summary

| Component           | Size       | Notes                           |
| ------------------- | ---------- | ------------------------------- |
| Posts (annual)      | 205 TB     | With 3x replication             |
| Feed cache (hot)    | 1 TB       | Active users                    |
| Feed cache (warm)   | 2 TB       | Recent users                    |
| Social graph        | 20 TB      | With indexes, replication       |
| **Total active**    | **~25 TB** | Plus historical posts           |

---

## Bandwidth Estimation

### Feed Read Bandwidth

**Per Feed Request:**
```
Posts returned: 20 per page
Post data: 750 bytes
Total per request: 20 × 750 = 15 KB

With metadata and overhead: ~20 KB per request
```

**Total Read Bandwidth:**
```
Peak requests: 1M QPS
Bandwidth = 1M × 20 KB = 20 GB/s = 160 Gbps
```

### Post Write Bandwidth

**Per Post:**
```
Post data: 750 bytes
Peak posts: 8,700/sec

Write bandwidth = 8,700 × 750 = 6.5 MB/s = 52 Mbps
```

### Fan-out Bandwidth (Internal)

**Feed Cache Updates:**
```
Fan-out writes: 573,000/sec
Per write: 20 bytes (post reference)

Internal bandwidth = 573,000 × 20 = 11.5 MB/s = 92 Mbps
```

### Total Bandwidth

| Direction       | Bandwidth | Notes                    |
| --------------- | --------- | ------------------------ |
| Feed reads      | 160 Gbps  | Peak user requests       |
| Post writes     | 52 Mbps   | Post creation            |
| Fan-out         | 92 Mbps   | Internal feed updates    |
| **Total**       | ~165 Gbps | Dominated by reads       |

---

## Memory Estimation

### Feed Cache (Redis)

**Hot Feed Cache:**
```
Active users: 100M
Feed size: 10 KB
Total: 100M × 10 KB = 1 TB

Distributed across Redis cluster:
- 64 GB per node
- Nodes needed: 1 TB / 64 GB = 16 nodes
- With replication (3x): 48 nodes
```

### Post Cache

**Recent Posts:**
```
Cache last 24 hours of posts
Posts: 250M
Post size: 750 bytes
Total: 250M × 750 = 187.5 GB

Redis nodes: 187.5 GB / 64 GB = 3 nodes
With replication: 9 nodes
```

### Social Graph Cache

**Active User Graph:**
```
Cache follows for active users
Active users: 100M
Follows per user: 200
Edge size: 8 bytes (just following_id)

Total: 100M × 200 × 8 = 160 GB
Redis nodes: 3 (with replication: 9)
```

### Total Memory

| Component           | Memory    | Redis Nodes |
| ------------------- | --------- | ----------- |
| Feed cache          | 1 TB      | 48          |
| Post cache          | 187.5 GB  | 9           |
| Graph cache         | 160 GB    | 9           |
| **Total**           | ~1.4 TB   | ~66 nodes   |

---

## Server Estimation

### Feed Service

**Calculation:**
```
Peak QPS: 1 million
QPS per server: 10,000 (with caching)

Feed servers = 1M / 10K = 100 servers
With redundancy: 150 servers
```

### Post Service

**Calculation:**
```
Peak writes: 8,700/sec
Writes per server: 2,000/sec

Post servers = 8,700 / 2,000 = 5 servers
With redundancy: 10 servers
```

### Fan-out Service

**Calculation:**
```
Fan-out writes: 573,000/sec
Writes per worker: 10,000/sec

Fan-out workers = 573,000 / 10,000 = 58 workers
With redundancy: 100 workers
```

### Ranking Service

**Calculation:**
```
Ranking requests: 1M/sec (same as feed)
Rankings per server: 5,000/sec (ML inference)

Ranking servers = 1M / 5K = 200 servers
```

### Server Summary

| Component           | Servers | Specs                           |
| ------------------- | ------- | ------------------------------- |
| Feed service        | 150     | 32 cores, 64 GB RAM             |
| Post service        | 10      | 16 cores, 32 GB RAM             |
| Fan-out workers     | 100     | 16 cores, 32 GB RAM             |
| Ranking service     | 200     | 32 cores, 128 GB RAM (ML)       |
| Redis cluster       | 66      | 16 cores, 64 GB RAM             |
| Kafka brokers       | 20      | 16 cores, 64 GB RAM, 2 TB SSD   |
| PostgreSQL          | 30      | 32 cores, 256 GB RAM, 10 TB SSD |
| **Total**           | **~576**|                                 |

---

## Cost Estimation

### Compute Costs (AWS)

| Component           | Instance Type | Count | Monthly Cost |
| ------------------- | ------------- | ----- | ------------ |
| Feed service        | c6i.8xlarge   | 150   | $150,000     |
| Post service        | c6i.4xlarge   | 10    | $6,000       |
| Fan-out workers     | c6i.4xlarge   | 100   | $60,000      |
| Ranking service     | p3.2xlarge    | 200   | $600,000     |
| Redis cluster       | r6i.2xlarge   | 66    | $24,000      |
| Kafka brokers       | r6i.2xlarge   | 20    | $7,200       |
| PostgreSQL          | r6i.8xlarge   | 30    | $45,000      |
| **Compute Total**   |               |       | **$892,200** |

### Storage Costs

| Type                | Size    | Monthly Cost |
| ------------------- | ------- | ------------ |
| PostgreSQL (SSD)    | 300 TB  | $30,000      |
| Redis (memory)      | 1.4 TB  | Included     |
| Kafka (SSD)         | 40 TB   | $4,000       |
| S3 (backups)        | 500 TB  | $11,500      |
| **Storage Total**   |         | **$45,500**  |

### Network Costs

```
Outbound: 165 Gbps peak = ~50 PB/month
At $0.05/GB = $2.5M/month

With CDN and optimizations: ~$500K/month
```

### Total Monthly Cost

| Category    | Cost         |
| ----------- | ------------ |
| Compute     | $892,200     |
| Storage     | $45,500      |
| Network     | $500,000     |
| Misc (20%)  | $287,540     |
| **Total**   | **~$1.7M/month** |

### Cost per User

```
Monthly cost: $1.7M
DAU: 500M

Cost per DAU = $1.7M / 500M = $0.0034/user/month
            = $0.04/user/year
```

---

## Scaling Projections

### 2x Growth (1B DAU)

| Metric          | Current     | 2x Scale    |
| --------------- | ----------- | ----------- |
| DAU             | 500M        | 1B          |
| Feed QPS        | 1M          | 2M          |
| Feed servers    | 150         | 300         |
| Redis nodes     | 66          | 130         |
| Monthly cost    | $1.7M       | $3.4M       |

### Key Scaling Challenges

1. **Feed Cache Size**: Grows linearly with users
   - Solution: More aggressive cache eviction, tiering

2. **Fan-out Volume**: Grows with users × follows
   - Solution: More fan-out workers, better batching

3. **Ranking Compute**: ML inference is expensive
   - Solution: Pre-compute rankings, use simpler models for tail

4. **Network Bandwidth**: Dominated by feed reads
   - Solution: CDN for static content, compression

---

## Quick Reference Numbers

### Interview Mental Math

```
500M DAU
1M peak QPS (feed reads)
250M posts/day
50B feed requests/day
573K fan-out writes/sec
1.4 TB memory (caches)
~600 servers
$1.7M/month
$0.04/user/year
```

### Key Ratios

```
Feed reads per post: 200 (average followers)
Feed requests per user per day: 60
Posts per user per day: 0.5
Celebrity threshold: 10K followers
```

---

## Summary

| Aspect              | Value                                  |
| ------------------- | -------------------------------------- |
| Peak feed QPS       | 1 million                              |
| Peak post writes    | 8,700/sec                              |
| Fan-out writes      | 573,000/sec                            |
| Feed cache size     | 1 TB (hot)                             |
| Total servers       | ~600                                   |
| Monthly cost        | ~$1.7M                                 |
| Cost per DAU        | $0.0034/month                          |

