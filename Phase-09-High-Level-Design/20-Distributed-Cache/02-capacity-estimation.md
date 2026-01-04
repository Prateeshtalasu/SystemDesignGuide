# Distributed Cache - Capacity Estimation

## Overview

This document calculates the infrastructure requirements for a distributed cache handling 100,000 requests per second with 1 TB of cached data.

---

## Traffic Estimation

### Request Rate

**Given:**
- Cache requests: 100K QPS
- Operating 24/7
- Hit rate: 80%

**Calculations:**

```
Cache requests per second = 100,000 QPS
Cache hits: 100K × 0.8 = 80K QPS
Cache misses: 100K × 0.2 = 20K QPS

Peak traffic (2x average during traffic spikes):
Peak QPS = 100K × 2 = 200K QPS
```

### Request Type Distribution

**Breakdown by operation:**

| Operation Type | % of Requests | QPS (Average) | QPS (Peak) |
|----------------|---------------|---------------|------------|
| GET (read)     | 90% | 90,000 | 180,000 |
| SET (write)    | 8% | 8,000 | 16,000 |
| DELETE         | 2% | 2,000 | 4,000 |
| **Total**      | **100%** | **100,000** | **200,000** |

---

## Storage Estimation

### Cache Data Storage

**Data Distribution:**
```
Total cache size: 1 TB
Data per key (average): 10 KB
Total keys: 1 TB / 10 KB = 100 million keys

Key distribution:
  - Hot keys (20%): 20M keys, accessed frequently
  - Warm keys (30%): 30M keys, accessed occasionally
  - Cold keys (50%): 50M keys, accessed rarely
```

**Storage per Node:**
```
Nodes: 10
Storage per node = 1 TB / 10 = 100 GB/node

With replication (2x):
Storage per node = 100 GB × 2 = 200 GB/node

**Math Verification:**
- Assumptions: 1 TB total cache, 10 nodes, 2x replication
- Per node: 1,000 GB / 10 = 100 GB
- With replication: 100 GB × 2 = 200 GB
- **DOC MATCHES:** Storage calculations verified ✅

### Key Metadata Storage

**Key Overhead:**
```
Keys: 100 million
Metadata per key:
  - Key string: 100 bytes (average)
  - TTL: 8 bytes
  - Access time: 8 bytes
  - Size: 8 bytes
  Total: ~124 bytes per key

Total metadata = 100M × 124 bytes
               = 12.4 GB

**Math Verification:**
- Assumptions: 100M keys, 124 bytes per key metadata
- Calculation: 100,000,000 × 124 bytes = 12,400,000,000 bytes = 12.4 GB
- **DOC MATCHES:** Storage calculations verified ✅

### Total Storage Summary

| Component           | Size       | Notes                           |
| ------------------- | ---------- | ------------------------------- |
| Cache data          | 1 TB       | Actual cached values            |
| Replication (2x)    | 2 TB       | With 2x replication              |
| Key metadata        | 12.4 GB    | Key overhead                    |
| **Total**           | **~2.01 TB**| Primary storage                 |

---

## Bandwidth Estimation

### Cache Read Bandwidth

**GET Operations:**
```
GET requests: 90K QPS (average)
Average value size: 10 KB
Peak multiplier: 2x

Average bandwidth = 90K × 10 KB/s
                  = 900 MB/s
                  = 7.2 Gbps

Peak bandwidth = 7.2 × 2 = 14.4 Gbps
```

### Cache Write Bandwidth

**SET Operations:**
```
SET requests: 8K QPS (average)
Average value size: 10 KB
Peak multiplier: 2x

Average bandwidth = 8K × 10 KB/s
                  = 80 MB/s
                  = 640 Mbps

Peak bandwidth = 640 × 2 = 1.28 Gbps
```

### Total Bandwidth

| Direction       | Bandwidth | Notes                    |
| --------------- | --------- | ------------------------ |
| Read (outbound) | 7.2 Gbps  | Cache GET responses      |
| Write (inbound) | 640 Mbps  | Cache SET requests       |
| **Total**       | **~8 Gbps** | Peak requirement         |

---

## Memory Estimation

### Per-Node Memory Requirements

**Cache Data:**
```
Storage per node: 200 GB (with replication)
Memory per node = 200 GB (all data in memory)
```

**Key Metadata:**
```
Keys per node: 100M / 10 = 10M keys/node
Metadata per key: 124 bytes
Metadata per node = 10M × 124 bytes
                  = 1.24 GB
```

**Redis Overhead:**
```
Redis internal structures: 20% overhead
Overhead per node = 200 GB × 0.2
                  = 40 GB
```

**Total Memory per Node:**
```
Cache data: 200 GB
Metadata: 1.24 GB
Overhead: 40 GB
Buffer (20%): 48 GB
Total: ~290 GB per node
```

### Total Memory Requirements

| Component              | Memory    | Notes                      |
| ---------------------- | --------- | -------------------------- |
| Per cache node (×10)   | 2.9 TB    | 290 GB × 10 nodes           |
| Application memory     | 32 GB     | Per application server      |
| **Total**              | **~2.93 TB**| Distributed across cluster |

---

## Server Estimation

### Cache Nodes

**Calculation:**
```
Cache requests: 100K QPS
Requests per node: 10,000/second

Nodes needed = 100K / 10K = 10 nodes
With redundancy: 15 nodes
```

**Cache Node Specs:**
```
CPU: 16 cores (handles high throughput)
Memory: 256 GB (for 200 GB cache data + overhead)
Network: 25 Gbps (high bandwidth requirement)
Disk: 1 TB SSD (AOF persistence)
```

### Application Servers

**Calculation:**
```
Cache clients: 50 application servers
Each server: 2K QPS to cache

Application servers: 50 (existing infrastructure)
```

### Server Summary

| Component           | Servers | Specs                           |
| ------------------- | ------- | ------------------------------- |
| Cache nodes         | 15      | 16 cores, 256 GB RAM, 1 TB SSD |
| Application servers | 50      | Existing infrastructure         |
| **Total**           | **15**  | Cache infrastructure only       |

---

## Cost Estimation

### Compute Costs (AWS)

| Component           | Instance Type | Count | Monthly Cost |
| ------------------- | ------------- | ----- | ------------ |
| Cache nodes         | r6i.8xlarge   | 15    | $9,000       |
| **Compute Total**   |               |       | **$9,000**   |

### Storage Costs

| Type                | Size/Requests | Monthly Cost |
| ------------------- | ------------- | ------------ |
| EBS (SSD)           | 15 TB         | $1,500       |
| **Storage Total**   |               | **$1,500**   |

### Network Costs

```
Data transfer: 8 Gbps peak = 2.6 TB/month
Data transfer cost: $230/month
Cross-AZ: ~$500/month

Network total: ~$730/month
```

### Total Monthly Cost

| Category    | Cost         |
| ----------- | ------------ |
| Compute     | $9,000      |
| Storage     | $1,500      |
| Network     | $730        |
| Misc (20%)  | $2,246      |
| **Total**   | **~$13,476/month** |

### Cost per Request

```
Monthly requests: 100K QPS × 86,400 × 30 = 259.2 billion requests/month
Monthly cost: $13,476

Cost per request = $13,476 / 259.2B
                 = $0.000000052
                 = $0.052 per million requests
```

### Cost Savings from Cache

**Database Load Reduction:**
```
Cache hits: 80K QPS (served from cache)
Database queries avoided: 80K QPS

Database cost per query: $0.001 (estimated)
Monthly savings = 80K × 86,400 × 30 × $0.001
                = $207,360/month

Net cost = $13,476 - $207,360 = -$193,884/month (savings!)
```

---

## Scaling Projections

### 10x Scale (1M QPS)

| Metric          | Current     | 10x Scale   |
| --------------- | ----------- | ----------- |
| QPS             | 100K        | 1M          |
| Storage         | 2 TB        | 20 TB       |
| Nodes           | 15          | 150         |
| Monthly cost    | $13.5K      | $135K       |

### Bottlenecks at Scale

1. **Memory requirements**: Linear growth with data size
   - Solution: Data compression, tiered storage

2. **Network bandwidth**: High bandwidth for large values
   - Solution: Regional deployment, CDN integration

3. **Key distribution**: Hot keys cause uneven load
   - Solution: Key replication, local caching

---

## Quick Reference Numbers

### Interview Mental Math

```
100,000 QPS
1 TB cache data
15 cache nodes
$13.5K/month
80% hit rate
$0.052 per million requests
```

### Key Ratios

```
Requests per node: 10K/second
Storage per node: 200 GB
Memory per node: 256 GB
Hit rate: 80%
Latency p95: < 1ms
```

---

## Summary

| Aspect              | Value                                  |
| ------------------- | -------------------------------------- |
| QPS                 | 100,000                                |
| Storage             | 2.01 TB                                |
| Memory              | 2.93 TB                                |
| Nodes               | 15                                     |
| Monthly cost        | ~$13,476                               |
| Cost per request    | $0.052 per million requests            |
| Hit rate            | 80%                                    |
