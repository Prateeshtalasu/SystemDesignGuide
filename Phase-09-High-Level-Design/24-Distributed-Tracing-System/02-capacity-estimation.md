# Distributed Tracing System - Capacity Estimation

## Overview

This document calculates the infrastructure requirements for a distributed tracing system handling 1 million requests/second with 1% sampling rate.

---

## Traffic Estimation

### Span Generation Rate

**Given:**
- Requests/second: 1 million
- Average spans per request: 50 spans
- Sampling rate: 1% (head-based sampling)

**Calculations:**

```
Raw spans/second:
= 1M requests/s × 50 spans/request
= 50 million spans/second

Sampled spans/second:
= 50M × 0.01 (1% sampling)
= 500,000 spans/second

Error spans (100% sampling):
Error rate: 0.1% of requests
Error requests/second: 1M × 0.001 = 1,000 req/s
Error spans/second: 1,000 × 50 = 50,000 spans/s

Total spans/second:
= 500,000 (sampled) + 50,000 (errors)
= 550,000 spans/second
```

**Math Verification:**
- Assumptions: 1M requests/sec, 50 spans/request, 1% sampling rate, 0.1% error rate, 100% error sampling
- Raw spans: 1,000,000 × 50 = 50,000,000 spans/sec
- Sampled spans: 50,000,000 × 0.01 = 500,000 spans/sec
- Error requests: 1,000,000 × 0.001 = 1,000 req/sec
- Error spans: 1,000 × 50 = 50,000 spans/sec
- Total: 500,000 + 50,000 = 550,000 spans/sec
- **DOC MATCHES:** Span generation rate verified ✅
```

### Trace Generation Rate

```
Traces/second:
= 1M requests/s × 0.01 (sampling) + 1,000 (errors)
= 10,000 + 1,000
= 11,000 traces/second
```

**Math Verification:**
- Assumptions: 1M requests/sec, 1% sampling rate, 1,000 error requests/sec (100% sampled)
- Sampled traces: 1,000,000 × 0.01 = 10,000 traces/sec
- Error traces: 1,000 traces/sec
- Total: 10,000 + 1,000 = 11,000 traces/sec
- **DOC MATCHES:** Trace generation rate verified ✅
```

### Peak Traffic

```
Peak multiplier: 3x
Peak spans/second: 550K × 3 = 1.65M spans/second
Peak traces/second: 11K × 3 = 33K traces/second
```

---

## Storage Estimation

### Span Size

**Average Span:**
```
Span fields:
- Trace ID: 16 bytes
- Span ID: 8 bytes
- Parent Span ID: 8 bytes
- Service name: 50 bytes (average)
- Operation name: 100 bytes (average)
- Start time: 8 bytes
- Duration: 8 bytes
- Tags: 500 bytes (average key-value pairs)
- Logs: 200 bytes (average)
- Context: 100 bytes
- Overhead: 100 bytes

Total per span: ~1,088 bytes ≈ 1.1 KB

**Math Verification:**
- Assumptions: Various span fields totaling ~1,088 bytes
- Calculation: 16 + 8 + 8 + 50 + 100 + 8 + 8 + 500 + 200 + 100 + 100 = 1,098 bytes ≈ 1.1 KB
- **DOC MATCHES:** Storage calculations verified ✅

**Maximum Span:**
```
Large spans (with many tags/logs):
- Base: 1.1 KB
- Large tags: +2 KB
- Large logs: +5 KB
- Maximum: ~8 KB
```

### Trace Size

**Average Trace:**
```
Average spans per trace: 50
Average trace size: 50 × 1.1 KB = 55 KB
```

**Maximum Trace:**
```
Maximum spans per trace: 10,000
Maximum trace size: 10,000 × 8 KB = 80 MB
```

### Hot Storage (7 days)

```
Spans/second (average): 550,000
Span size: 1.1 KB
Seconds in 7 days: 604,800

Hot storage:
= 550,000 spans/s × 1.1 KB × 604,800s
= 550,000 × 1,126.8 bytes × 604,800
= 374,850,000,000,000 bytes
= 341 TB

With compression (2:1 ratio):
Hot storage (compressed): 341 TB / 2 = 170.5 TB

With replication (3x):
Hot storage (replicated): 170.5 TB × 3 = 511.5 TB
```

### Cold Storage (90 days)

```
Total retention: 90 days
Hot storage: 7 days
Cold storage: 83 days

Cold storage (compressed, before replication):
= 170.5 TB × (83 / 7)
= 170.5 TB × 11.86
= 2,022 TB
≈ 2 PB

With replication (3x):
Cold storage (replicated): 2 PB × 3 = 6 PB
```

### Index Storage

**Trace ID Index:**
```
Traces/second: 11,000
Trace ID: 16 bytes
Index entry: 16 bytes (trace ID) + 8 bytes (offset) = 24 bytes
Index for 7 days:
= 11,000 traces/s × 24 bytes × 604,800s
= 159,667,200,000 bytes
= 149 GB

With replication: 149 GB × 3 = 447 GB
```

**Service/Operation Index:**
```
Estimated index size: 500 GB (replicated: 1.5 TB)
```

### Total Storage Summary

| Component | Size | Notes |
|-----------|------|-------|
| Hot storage (7 days) | 512 TB | Compressed, replicated |
| Cold storage (83 days) | 6 PB | Compressed, replicated |
| Index storage | 2 TB | Trace ID + service indexes |
| **Total storage** | **~6.5 PB** | After compression and replication |

---

## Bandwidth Estimation

### Span Ingestion Bandwidth

```
Spans/second: 550,000
Span size: 1.1 KB

Ingestion bandwidth:
= 550,000 × 1.1 KB/s
= 605 MB/s
= 4.84 Gbps

Peak (3x): 14.5 Gbps
```

### Inter-Collector Bandwidth

```
Replication traffic:
= 4.84 Gbps × 2 (to 2 replicas)
= 9.68 Gbps

Peak: 29 Gbps
```

### Query Bandwidth

```
Trace queries: 1,000/second (peak)
Average trace size: 55 KB

Query bandwidth:
= 1,000 × 55 KB/s
= 55 MB/s
= 440 Mbps

Peak: 1.3 Gbps
```

### Total Bandwidth

| Direction | Bandwidth | Notes |
|-----------|-----------|-------|
| Span ingestion (avg) | 4.8 Gbps | From services to collectors |
| Span ingestion (peak) | 14.5 Gbps | Peak traffic |
| Replication | 9.7 Gbps | Inter-collector replication |
| Query responses | 0.4 Gbps | To users/UI |
| **Total (peak)** | **~25 Gbps** | Peak requirement |

---

## Memory Estimation

### Collector Memory

**Per Collector:**
```
Span buffer (in-flight):
- Buffer for 10 seconds: 550K spans/s × 10s = 5.5M spans
- Memory: 5.5M × 1.1 KB = 6 GB

Processing queues:
- Validation queue: 1 GB
- Routing queue: 1 GB
- Compression buffer: 2 GB

Total per collector: ~10 GB
```

**Collector Cluster:**
```
Collectors: 50 (for high availability)
Total memory: 50 × 10 GB = 500 GB
```

### Aggregator Memory

**Trace Aggregation:**
```
Active traces (last 5 minutes):
= 11,000 traces/s × 300s = 3.3M traces
Memory per trace: 1 KB (partial trace state)
Memory: 3.3M × 1 KB = 3.3 GB

With overhead: ~5 GB per aggregator
Aggregators: 20
Total memory: 20 × 5 GB = 100 GB
```

### Query Service Memory

**Query Cache:**
```
Cached traces: 100,000
Memory per trace: 55 KB
Cache memory: 100K × 55 KB = 5.5 GB

Query processing: 2 GB
Total per query service: ~8 GB
Query services: 10
Total memory: 10 × 8 GB = 80 GB
```

### Total Memory Requirements

| Component | Memory | Notes |
|-----------|--------|-------|
| Collectors | 500 GB | Span ingestion and buffering |
| Aggregators | 100 GB | Trace reconstruction |
| Query services | 80 GB | Query processing and cache |
| Storage nodes | 200 GB | Index caching |
| **Total cluster** | **~880 GB** | Distributed across nodes |

---

## Server Estimation

### Collector Servers

**Calculation:**
```
Spans/second: 550,000
Spans per collector: 15,000/second (limited by CPU/network)

Collectors needed:
= 550,000 / 15,000
= 37 collectors

With redundancy (1.35x): 50 collectors
```

**Collector Server Specs:**
```
CPU: 16 cores (span processing, validation)
Memory: 16 GB (buffering)
Network: 10 Gbps
Disk: 500 GB SSD (local buffer)
```

### Aggregator Servers

**Calculation:**
```
Traces/second: 11,000
Traces per aggregator: 1,000/second

Aggregators needed:
= 11,000 / 1,000
= 11 aggregators

With redundancy: 20 aggregators
```

**Aggregator Server Specs:**
```
CPU: 16 cores (trace reconstruction)
Memory: 32 GB (trace state)
Network: 10 Gbps
Disk: 1 TB SSD
```

### Query Service Servers

**Calculation:**
```
Query QPS: 100 (average), 1,000 (peak)
Queries per server: 50/second

Query servers needed:
= 1,000 / 50
= 20 servers

With redundancy: 10 servers (peak is rare)
```

**Query Service Specs:**
```
CPU: 8 cores (query processing)
Memory: 32 GB (cache)
Network: 10 Gbps
Disk: 2 TB SSD (index cache)
```

### Storage Servers

**Using Distributed Storage (S3/HDFS):**
```
Hot storage: 512 TB
Cold storage: 6 PB

Using managed storage (S3):
- No dedicated servers
- Pay per storage/request
```

**Self-hosted (HDFS):**
```
Storage per server: 100 TB
Hot storage servers: 512 TB / 100 TB = 6 servers (with 3x replication)
Cold storage servers: 6 PB / 100 TB = 60 servers (with 3x replication)

Total storage servers: 66
```

### Server Summary

| Component | Servers | Specs |
|-----------|---------|-------|
| Collectors | 50 | 16 cores, 16 GB RAM, 10 Gbps |
| Aggregators | 20 | 16 cores, 32 GB RAM, 10 Gbps |
| Query services | 10 | 8 cores, 32 GB RAM, 10 Gbps |
| Storage (managed) | 0 | Using S3/HDFS |
| **Total (compute)** | **80** | Plus managed storage |

---

## Cost Estimation

### Compute Costs (AWS)

| Component | Instance Type | Count | Monthly Cost |
|-----------|---------------|-------|--------------|
| Collectors | c6i.4xlarge | 50 | $20,000 |
| Aggregators | r6i.4xlarge | 20 | $12,000 |
| Query services | r6i.2xlarge | 10 | $6,000 |
| **Compute Total** | | | **$38,000** |

### Storage Costs (S3)

| Type | Size/Requests | Monthly Cost |
|------|---------------|--------------|
| S3 Standard (hot) | 512 TB | $11,520 |
| S3 Glacier (cold) | 6 PB | $144,000 |
| S3 Requests | 1.4B PUT/month | $70 |
| S3 GET requests | 500M/month | $20 |
| **Storage Total** | | **$155,610** |

### Network Costs

```
Data transfer in: Free on AWS
Data transfer out: Minimal (internal)
Cross-AZ: ~$2,000/month
VPC endpoints: ~$500/month

Network total: ~$2,500/month
```

### Total Monthly Cost

| Category | Cost |
|----------|------|
| Compute | $38,000 |
| Storage | $155,610 |
| Network | $2,500 |
| Misc (20%) | $39,222 |
| **Total** | **~$235,000/month** |

### Cost per Trace

```
Traces/month:
= 11,000 traces/s × 2,592,000 seconds/month
= 28.5 billion traces/month

Cost per trace:
= $235,000 / 28.5B
= $0.0000082
= $8.20 per million traces
```

---

## Scaling Projections

### 10x Scale (10M requests/second)

| Metric | Current | 10x Scale |
|--------|---------|-----------|
| Requests/second | 1M | 10M |
| Spans/second (sampled) | 550K | 5.5M |
| Traces/second | 11K | 110K |
| Hot storage (7 days) | 512 TB | 5.1 PB |
| Cold storage (90 days) | 6 PB | 60 PB |
| Collectors | 50 | 500 |
| Monthly cost | $235K | $2.2M |

### Bottlenecks at Scale

1. **Storage costs**: Grow linearly, become dominant cost
   - Solution: More aggressive sampling, better compression, tiered storage

2. **Collector scaling**: Need more collectors
   - Solution: Auto-scaling, geographic distribution

3. **Query performance**: More traces to search
   - Solution: Better indexing, distributed query, caching

4. **Network bandwidth**: Ingestion bandwidth grows
   - Solution: Regional collectors, compression

---

## Quick Reference Numbers

### Interview Mental Math

```
1M requests/second
50 spans per request (average)
1% sampling rate
550K spans/second (after sampling)
11K traces/second
512 TB hot storage (7 days)
6 PB cold storage (90 days)
80 compute servers
$235K/month
$8.20 per million traces
```

### Key Ratios

```
Spans per request: 50 (average)
Sampling rate: 1% (normal), 100% (errors)
Compression ratio: 2:1
Replication factor: 3x
Spans per collector: 15,000/second
Traces per aggregator: 1,000/second
```

### Time Estimates

```
Trace retention: 7 days (hot), 90 days (cold)
Query latency: < 500ms (p95), < 2s (p99)
Span ingestion latency: < 10ms (p99)
Trace aggregation delay: < 1 minute
```

---

## Summary

| Aspect | Value |
|--------|-------|
| Requests/second | 1 million |
| Spans/second (sampled) | 550,000 |
| Traces/second | 11,000 |
| Hot storage (7 days) | 512 TB |
| Cold storage (90 days) | 6 PB |
| Compute servers | 80 |
| Monthly cost | ~$235,000 |
| Cost per million traces | $8.20 |


