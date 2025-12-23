# Web Crawler - Capacity Estimation

## Overview

This document calculates the infrastructure requirements for a web crawler handling 1 billion pages per month.

---

## Traffic Estimation

### Crawl Rate

**Given:**
- Target: 1 billion pages/month
- Operating 24/7

**Calculations:**

```
Pages per day = 1B / 30 days
             = 33.3 million pages/day

Pages per hour = 33.3M / 24 hours
              = 1.39 million pages/hour

Pages per second = 1.39M / 3,600 seconds
                = 386 pages/second

With 3x buffer for peak/catch-up:
Target crawl rate = 386 × 3 = ~1,000 pages/second
```

### DNS Lookups

**DNS Query Rate:**
```
Unique domains: ~100 million (estimated)
New domains per second: ~100 (as we discover new sites)

With caching (1 hour TTL):
Cache hit rate: ~90%
DNS queries/second = 1,000 × 0.1 = 100 queries/second
```

### HTTP Connections

**Connection Requirements:**
```
Pages/second: 1,000
Average page download time: 2 seconds
Concurrent connections needed: 1,000 × 2 = 2,000

With connection reuse (keep-alive):
Actual new connections/second: ~500
```

---

## Storage Estimation

### Raw Page Storage

**Average Page Size:**
```
Raw HTML: 100 KB average
After gzip compression: 20 KB average
Headers/metadata: 1 KB
Total per page: ~21 KB compressed
```

**Monthly Storage:**
```
Pages per month: 1 billion
Storage per page: 21 KB

Monthly storage = 1B × 21 KB
               = 21 TB/month
```

**Annual Storage:**
```
Annual storage = 21 TB × 12 months
              = 252 TB/year

With 3x replication = 756 TB/year
```

### URL Frontier Storage

**Frontier Size:**
```
URLs in queue: 100 million (buffer)
Bytes per URL entry:
  - URL hash: 8 bytes
  - Priority: 4 bytes
  - Timestamp: 8 bytes
  - Domain ID: 8 bytes
  - Status: 1 byte
  Total: ~30 bytes

Frontier storage = 100M × 30 bytes = 3 GB
```

### Seen URLs (Bloom Filter)

**Bloom Filter for Deduplication:**
```
URLs seen: 10 billion (historical)
False positive rate: 0.1%
Bits per element: ~14.4 bits

Bloom filter size = 10B × 14.4 bits / 8
                 = 18 GB
```

### Robots.txt Cache

**Cache Size:**
```
Unique domains: 100 million
Average robots.txt: 2 KB
Cache all: 100M × 2 KB = 200 GB

With LRU (keep top 10M):
Cache size = 10M × 2 KB = 20 GB
```

### Total Storage Summary

| Component           | Size       | Notes                           |
| ------------------- | ---------- | ------------------------------- |
| Raw pages (monthly) | 21 TB      | Compressed HTML                 |
| Raw pages (annual)  | 252 TB     | Before replication              |
| URL frontier        | 3 GB       | In-memory queue                 |
| Bloom filter        | 18 GB      | Seen URL deduplication          |
| Robots.txt cache    | 20 GB      | Top 10M domains                 |
| DNS cache           | 5 GB       | Domain → IP mappings            |
| **Active memory**   | **~50 GB** | Per crawler cluster             |

---

## Bandwidth Estimation

### Download Bandwidth

**Page Downloads:**
```
Pages/second: 1,000
Average page size: 100 KB (uncompressed)
With compression: 20 KB

Download bandwidth = 1,000 × 100 KB/s
                  = 100 MB/s
                  = 800 Mbps

Peak (3x): 2.4 Gbps
```

### Upload Bandwidth (Requests)

**HTTP Requests:**
```
Request size: ~500 bytes (headers + URL)
Requests/second: 1,000

Upload bandwidth = 1,000 × 500 bytes/s
                = 500 KB/s
                = 4 Mbps
```

### Storage Write Bandwidth

**To Document Store:**
```
Compressed pages: 1,000/sec × 21 KB = 21 MB/s
To S3/HDFS: 21 MB/s = 168 Mbps
```

### Total Bandwidth

| Direction       | Bandwidth | Notes                    |
| --------------- | --------- | ------------------------ |
| Download (avg)  | 800 Mbps  | Page downloads           |
| Download (peak) | 2.4 Gbps  | 3x buffer                |
| Upload          | 4 Mbps    | HTTP requests            |
| Storage write   | 168 Mbps  | To document store        |
| **Total**       | ~3 Gbps   | Peak requirement         |

---

## Memory Estimation

### Per-Crawler Memory

**Components:**
```
HTTP connection pool: 100 MB (2,000 connections × 50 KB)
DNS cache: 500 MB
Robots.txt cache: 2 GB
Parse buffers: 500 MB
Bloom filter (local): 2 GB

Total per crawler: ~5 GB
```

### Frontier Memory

**URL Frontier:**
```
In-memory frontier: 3 GB
Priority queues: 1 GB
Domain queues: 2 GB

Total frontier: ~6 GB
```

### Total Memory Requirements

| Component              | Memory    | Notes                      |
| ---------------------- | --------- | -------------------------- |
| Per crawler (×10)      | 50 GB     | 10 crawler instances       |
| URL frontier           | 6 GB      | Centralized queue          |
| Coordinator            | 4 GB      | State management           |
| **Total cluster**      | **60 GB** |                            |

---

## Server Estimation

### Crawler Servers

**Calculation:**
```
Target: 1,000 pages/second
Pages per crawler: 100/second (limited by network/CPU)

Crawlers needed = 1,000 / 100 = 10 crawlers
With redundancy: 15 crawlers
```

**Crawler Server Specs:**
```
CPU: 8 cores (HTTP handling, parsing)
Memory: 8 GB
Network: 1 Gbps
Disk: 100 GB SSD (local cache)
```

### DNS Resolver Servers

**Calculation:**
```
DNS queries: 100/second
Queries per resolver: 1,000/second

Resolvers needed: 2 (with redundancy)
```

### Coordinator Servers

**Calculation:**
```
Manage URL frontier
Coordinate 15 crawlers
Track progress

Coordinators needed: 3 (HA cluster)
```

### Storage Servers

**For Document Store (S3/HDFS):**
```
Using managed service (S3):
- No dedicated servers
- Pay per storage/request

Self-hosted HDFS:
- 252 TB/year × 3 replication = 756 TB
- 10 TB per server = 76 servers
```

### Server Summary

| Component           | Servers | Specs                           |
| ------------------- | ------- | ------------------------------- |
| Crawlers            | 15      | 8 cores, 8 GB RAM, 1 Gbps       |
| DNS resolvers       | 2       | 4 cores, 4 GB RAM               |
| Coordinators        | 3       | 8 cores, 16 GB RAM              |
| Kafka brokers       | 3       | 8 cores, 32 GB RAM, 1 TB SSD    |
| **Total**           | **23**  | Plus managed storage            |

---

## Cost Estimation

### Compute Costs (AWS)

| Component           | Instance Type | Count | Monthly Cost |
| ------------------- | ------------- | ----- | ------------ |
| Crawlers            | c6i.2xlarge   | 15    | $3,600       |
| DNS resolvers       | c6i.xlarge    | 2     | $240         |
| Coordinators        | r6i.2xlarge   | 3     | $1,080       |
| Kafka brokers       | r6i.2xlarge   | 3     | $1,080       |
| **Compute Total**   |               |       | **$6,000**   |

### Storage Costs

| Type                | Size/Requests | Monthly Cost |
| ------------------- | ------------- | ------------ |
| S3 Standard         | 21 TB/month   | $480         |
| S3 Requests         | 1B PUT        | $5,000       |
| S3 (annual growth)  | 252 TB        | $5,800       |
| **Storage Total**   |               | **$6,000**   |

### Network Costs

```
Data transfer out: minimal (crawling is inbound)
Data transfer in: free on AWS
Cross-AZ: ~$500/month

Network total: ~$500/month
```

### Total Monthly Cost

| Category    | Cost         |
| ----------- | ------------ |
| Compute     | $6,000       |
| Storage     | $6,000       |
| Network     | $500         |
| Misc (20%)  | $2,500       |
| **Total**   | **~$15,000/month** |

### Cost per Page

```
Monthly pages: 1 billion
Monthly cost: $15,000

Cost per page = $15,000 / 1B = $0.000015
            = $15 per million pages
```

---

## Scaling Projections

### 10x Scale (10B pages/month)

| Metric          | Current     | 10x Scale   |
| --------------- | ----------- | ----------- |
| Pages/month     | 1B          | 10B         |
| Crawl rate      | 1,000/sec   | 10,000/sec  |
| Crawlers        | 15          | 150         |
| Storage/month   | 21 TB       | 210 TB      |
| Monthly cost    | $15K        | $120K       |

### Bottlenecks at Scale

1. **Politeness limits**: Can't crawl same domain faster
   - Solution: Crawl more domains in parallel

2. **DNS resolution**: More unique domains
   - Solution: More DNS resolvers, longer cache TTL

3. **Coordination overhead**: More crawlers to manage
   - Solution: Hierarchical coordination, sharding by domain

4. **Storage costs**: Linear growth
   - Solution: Better compression, deduplication

---

## Quick Reference Numbers

### Interview Mental Math

```
1 billion pages/month
1,000 pages/second
21 TB storage/month
15 crawler servers
$15K/month
$15 per million pages
```

### Key Ratios

```
Pages per crawler: 100/second
Compressed page size: 21 KB
DNS cache hit rate: 90%
Bandwidth per crawler: 80 Mbps
```

### Time Estimates

```
Crawl 1M pages: ~17 minutes
Crawl 100M pages: ~28 hours
Full re-crawl (1B): ~12 days
```

---

## Summary

| Aspect              | Value                                  |
| ------------------- | -------------------------------------- |
| Crawl rate          | 1,000 pages/second                     |
| Monthly pages       | 1 billion                              |
| Storage/month       | 21 TB (compressed)                     |
| Bandwidth           | 3 Gbps peak                            |
| Crawler servers     | 15                                     |
| Total servers       | ~23                                    |
| Monthly cost        | ~$15,000                               |
| Cost per page       | $0.000015                              |

