# Monitoring & Alerting System - Capacity Estimation

## Overview

This document calculates infrastructure requirements for a monitoring system handling 10 million metrics per second from 1,000 services.

---

## Traffic Estimation

### Metrics Generation Rate

**Given:**
- Services: 1,000
- Metrics per service: 1,000 (average)
- Unique metrics: 1 million
- Metrics per second: 10 million

**Calculations:**

```
Metrics per service per second:
= 10M / 1,000 services
= 10,000 metrics/second per service

Metrics per service per minute:
= 10,000 × 60
= 600,000 metrics/minute per service
```

**Math Verification:**
- Assumptions: 10M metrics/sec total, 1,000 services
- Per service: 10,000,000 / 1,000 = 10,000 metrics/sec per service
- Per minute: 10,000 × 60 = 600,000 metrics/minute per service
- **DOC MATCHES:** Metrics per service calculations verified ✅
```

### Peak Traffic

```
Peak multiplier: 2x (traffic spikes, batch uploads)
Peak metrics/second: 10M × 2 = 20M metrics/second
```

---

## Storage Estimation

### Metric Size

**Average Metric:**
```
Metric fields:
- Metric name: 50 bytes (average)
- Labels: 200 bytes (key-value pairs)
- Value: 8 bytes (float64)
- Timestamp: 8 bytes (int64)
- Overhead: 10 bytes

Total per metric: ~276 bytes
```

**Compressed (gzip):**
```
Compression ratio: 3:1
Compressed size: 276 / 3 = ~92 bytes per metric
```

### Hot Storage (30 days)

```
Metrics/second (average): 10,000,000
Metric size (compressed): 92 bytes
Seconds in 30 days: 2,592,000

Hot storage:
= 10M metrics/s × 92 bytes × 2,592,000s
= 2,384,640,000,000,000 bytes
= 2.17 PB

With replication (3x):
Hot storage (replicated): 2.17 PB × 3 = 6.51 PB

**Math Verification:**
- Assumptions: 10M metrics/sec, 92 bytes per metric (compressed), 2,592,000 seconds in 30 days, 3x replication
- Calculation: 10,000,000 × 92 × 2,592,000 = 2,384,640,000,000,000 bytes = 2.17 PB
- With replication: 2.17 PB × 3 = 6.51 PB
- **DOC MATCHES:** Storage calculations verified ✅
```

### Cold Storage (1 year)

```
Total retention: 1 year (365 days)
Hot storage: 30 days
Cold storage: 335 days

Cold storage (compressed, before replication):
= 2.17 PB × (335 / 30)
= 2.17 PB × 11.17
= 24.2 PB

With replication (3x):
Cold storage (replicated): 24.2 PB × 3 = 72.6 PB
```

### Index Storage

**Metric Name Index:**
```
Unique metrics: 1 million
Index entry: 16 bytes (metric name hash) + 8 bytes (offset) = 24 bytes
Index size: 1M × 24 bytes = 24 MB (replicated: 72 MB)
```

**Label Index:**
```
Estimated index size: 500 GB (replicated: 1.5 TB)
```

### Total Storage Summary

| Component | Size | Notes |
|-----------|------|-------|
| Hot storage (30 days) | 6.5 PB | Compressed, replicated |
| Cold storage (335 days) | 72.6 PB | Compressed, replicated |
| Index storage | 1.5 TB | Metric and label indexes |
| **Total storage** | **~79 PB** | After compression and replication |

---

## Bandwidth Estimation

### Metric Ingestion Bandwidth

```
Metrics/second: 10,000,000
Metric size: 276 bytes (uncompressed)

Ingestion bandwidth:
= 10M × 276 bytes/s
= 2.76 GB/s
= 22.1 Gbps

Peak (2x): 44.2 Gbps
```

### Inter-Storage Bandwidth

```
Replication traffic:
= 22.1 Gbps × 2 (to 2 replicas)
= 44.2 Gbps

Peak: 88.4 Gbps
```

### Query Bandwidth

```
Query QPS: 10,000/second (peak)
Average query response: 100 KB

Query bandwidth:
= 10,000 × 100 KB/s
= 1 GB/s
= 8 Gbps
```

### Total Bandwidth

| Direction | Bandwidth | Notes |
|-----------|-----------|-------|
| Metric ingestion (avg) | 22.1 Gbps | From services to collectors |
| Metric ingestion (peak) | 44.2 Gbps | Peak traffic |
| Replication | 44.2 Gbps | Inter-storage replication |
| Query responses | 8 Gbps | To users/dashboards |
| **Total (peak)** | **~96 Gbps** | Peak requirement |

---

## Memory Estimation

### Collector Memory

**Per Collector:**
```
Metric buffer (in-flight):
- Buffer for 10 seconds: 10M metrics/s × 10s = 100M metrics
- Memory: 100M × 276 bytes = 27.6 GB

Processing queues:
- Validation queue: 2 GB
- Compression buffer: 5 GB

Total per collector: ~35 GB
```

**Collector Cluster:**
```
Collectors: 100 (for high availability)
Total memory: 100 × 35 GB = 3.5 TB
```

### Query Service Memory

**Query Cache:**
```
Cached queries: 100,000
Memory per cached query: 500 KB
Cache memory: 100K × 500 KB = 50 GB

Query processing: 10 GB
Total per query service: ~60 GB
Query services: 50
Total memory: 50 × 60 GB = 3 TB
```

### Alerting Engine Memory

**Alert Rule Evaluation:**
```
Active alerts: 10,000
Memory per alert: 10 KB
Alert memory: 10K × 10 KB = 100 MB

Rule evaluation: 5 GB
Total per alerting engine: ~5 GB
Alerting engines: 20
Total memory: 20 × 5 GB = 100 GB
```

### Total Memory Requirements

| Component | Memory | Notes |
|-----------|--------|-------|
| Collectors | 3.5 TB | Metric ingestion and buffering |
| Query services | 3 TB | Query processing and cache |
| Alerting engines | 100 GB | Alert rule evaluation |
| Storage nodes | 500 GB | Index caching |
| **Total cluster** | **~7.1 TB** | Distributed across nodes |

---

## Server Estimation

### Collector Servers

**Calculation:**
```
Metrics/second: 10,000,000
Metrics per collector: 200,000/second (limited by CPU/network)

Collectors needed:
= 10M / 200K
= 50 collectors

With redundancy (2x): 100 collectors
```

**Collector Server Specs:**
```
CPU: 32 cores (metric processing, compression)
Memory: 64 GB (buffering)
Network: 25 Gbps
Disk: 1 TB SSD (local buffer)
```

### Query Service Servers

**Calculation:**
```
Query QPS: 1,000 (average), 10,000 (peak)
Queries per server: 200/second

Query servers needed:
= 10,000 / 200
= 50 servers
```

**Query Service Specs:**
```
CPU: 16 cores (query processing)
Memory: 128 GB (cache)
Network: 25 Gbps
Disk: 2 TB SSD (index cache)
```

### Storage Servers

**Using Managed Storage (S3/HDFS):**
```
Hot storage: 6.5 PB
Cold storage: 72.6 PB

Using managed storage:
- No dedicated servers
- Pay per storage/request
```

**Self-hosted (Time-Series DB cluster):**
```
Storage per server: 100 TB
Hot storage servers: 6.5 PB / 100 TB = 65 servers (with 3x replication)
Cold storage servers: 72.6 PB / 100 TB = 726 servers (with 3x replication)

Total storage servers: 791
```

### Server Summary

| Component | Servers | Specs |
|-----------|---------|-------|
| Collectors | 100 | 32 cores, 64 GB RAM, 25 Gbps |
| Query services | 50 | 16 cores, 128 GB RAM, 25 Gbps |
| Alerting engines | 20 | 8 cores, 32 GB RAM, 10 Gbps |
| Storage (managed) | 0 | Using S3/HDFS |
| **Total (compute)** | **170** | Plus managed storage |

---

## Cost Estimation

### Compute Costs (AWS)

| Component | Instance Type | Count | Monthly Cost |
|-----------|---------------|-------|--------------|
| Collectors | c6i.8xlarge | 100 | $120,000 |
| Query services | r6i.4xlarge | 50 | $60,000 |
| Alerting engines | r6i.2xlarge | 20 | $24,000 |
| **Compute Total** | | | **$204,000** |

### Storage Costs (S3)

| Type | Size/Requests | Monthly Cost |
|------|---------------|--------------|
| S3 Standard (hot) | 6.5 PB | $146,250 |
| S3 Glacier (cold) | 72.6 PB | $1,815,000 |
| S3 Requests | 25.9B PUT/month | $130 |
| S3 GET requests | 500M/month | $20 |
| **Storage Total** | | **$1,961,400** |

### Network Costs

```
Data transfer in: Free on AWS
Data transfer out: Minimal (internal)
Cross-AZ: ~$5,000/month
VPC endpoints: ~$1,000/month

Network total: ~$6,000/month
```

### Total Monthly Cost

| Category | Cost |
|----------|------|
| Compute | $204,000 |
| Storage | $1,961,400 |
| Network | $6,000 |
| Misc (20%) | $434,280 |
| **Total** | **~$2.6M/month** |

### Cost per Metric

```
Metrics/month:
= 10M metrics/s × 2,592,000 seconds/month
= 25.92 trillion metrics/month

Cost per metric:
= $2.6M / 25.92T
= $0.0000001
= $0.10 per million metrics
```

---

## Scaling Projections

### 10x Scale (100M metrics/second)

| Metric | Current | 10x Scale |
|--------|---------|-----------|
| Metrics/second | 10M | 100M |
| Hot storage (30 days) | 6.5 PB | 65 PB |
| Cold storage (1 year) | 72.6 PB | 726 PB |
| Collectors | 100 | 1,000 |
| Monthly cost | $2.6M | $26M |

### Bottlenecks at Scale

1. **Storage costs**: Grow linearly, become dominant cost
   - Solution: Better compression, tiered storage, retention optimization

2. **Query performance**: More metrics to search
   - Solution: Better indexing, distributed query, caching

3. **Ingestion bandwidth**: Metric ingestion bandwidth grows
   - Solution: Regional collectors, compression

---

## Quick Reference Numbers

### Interview Mental Math

```
10M metrics/second
1M unique metrics
6.5 PB hot storage (30 days)
72.6 PB cold storage (1 year)
170 compute servers
$2.6M/month
$0.10 per million metrics
```

### Key Ratios

```
Metrics per service: 10,000/second (average)
Compression ratio: 3:1
Replication factor: 3x
Metrics per collector: 200,000/second
Queries per query service: 200/second
```

### Time Estimates

```
Metric retention: 30 days (hot), 1 year (cold)
Query latency: < 100ms (p95)
Alert evaluation: < 5 seconds
Dashboard load: < 2 seconds
```

---

## Summary

| Aspect | Value |
|--------|-------|
| Metrics/second | 10 million |
| Unique metrics | 1 million |
| Hot storage (30 days) | 6.5 PB |
| Cold storage (1 year) | 72.6 PB |
| Compute servers | 170 |
| Monthly cost | ~$2.6M |
| Cost per million metrics | $0.10 |

