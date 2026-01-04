# Distributed Rate Limiter - Capacity Estimation

## Overview

This document calculates the infrastructure requirements for a distributed rate limiter handling 100,000 requests per second across multiple API servers.

---

## Traffic Estimation

### Request Rate

**Given:**
- Total API requests: 100K QPS
- All requests need rate limiting
- Operating 24/7

**Calculations:**

```
Rate limit checks per second = 100,000 QPS
Peak traffic (2x average during traffic spikes):
Peak QPS = 100K × 2 = 200K QPS
```

### Request Type Distribution

**Breakdown by operation:**

| Operation Type | % of Requests | QPS (Average) | QPS (Peak) |
|----------------|---------------|---------------|------------|
| Rate limit check | 100% | 100,000 | 200,000 |
| Counter increment | 100% | 100,000 | 200,000 |
| Counter read | 100% | 100,000 | 200,000 |

---

## Storage Estimation

### Rate Limit Counter Storage

**Counter Data:**
```
Active rate limit keys: 10 million (unique user+endpoint combinations)
Data per counter:
  - Key: 100 bytes (user_id:endpoint)
  - Count: 8 bytes (integer)
  - Last reset: 8 bytes (timestamp)
  Total: ~116 bytes per counter

Active counter storage = 10M × 116 bytes
                      = 1.16 GB

**Math Verification:**
- Assumptions: 10M active counters, 116 bytes per counter
- Calculation: 10,000,000 × 116 bytes = 1,160,000,000 bytes = 1.16 GB
- **DOC MATCHES:** Storage calculations verified ✅

**Token Bucket Storage:**
```
Active buckets: 10 million
Data per bucket:
  - Key: 100 bytes
  - Tokens: 8 bytes
  - Last refill: 8 bytes
  - Capacity: 8 bytes
  - Refill rate: 8 bytes
  Total: ~132 bytes per bucket

Active bucket storage = 10M × 132 bytes
                      = 1.32 GB

**Math Verification:**
- Assumptions: 10M active buckets, 132 bytes per bucket
- Calculation: 10,000,000 × 132 bytes = 1,320,000,000 bytes = 1.32 GB
- **DOC MATCHES:** Storage calculations verified ✅

**Sliding Window Storage:**
```
Active windows: 10 million
Data per window (Redis sorted set):
  - Key: 100 bytes
  - Members: Variable (request timestamps)
  - Average requests per window: 100
  - Data per member: 16 bytes (timestamp + request_id)
  Total per window: 100 + (100 × 16) = 1,700 bytes

Active window storage = 10M × 1,700 bytes
                      = 17 GB

**Math Verification:**
- Assumptions: 10M active windows, 1,700 bytes per window
- Calculation: 10,000,000 × 1,700 bytes = 17,000,000,000 bytes = 17 GB
- **DOC MATCHES:** Storage calculations verified ✅

### Total Storage Summary

| Component           | Size       | Notes                           |
| ------------------- | ---------- | ------------------------------- |
| Rate limit counters | 1.16 GB    | Active counters                  |
| Token buckets       | 1.32 GB    | Active buckets                  |
| Sliding windows     | 17 GB      | Active windows (sorted sets)    |
| **Total**           | **~20 GB** | In-memory (Redis)                |

---

## Bandwidth Estimation

### Redis Operations

**Incoming Operations:**
```
Operations per second: 100K
Average operation size: 200 bytes (GET/SET commands)
Peak multiplier: 2x

Average bandwidth = 100K × 200 bytes/s
                  = 20 MB/s
                  = 160 Mbps

Peak bandwidth = 160 × 2 = 320 Mbps
```

### Response Bandwidth

**Outgoing Responses:**
```
Responses per second: 100K
Average response size: 50 bytes (OK/error)
Peak multiplier: 2x

Average bandwidth = 100K × 50 bytes/s
                  = 5 MB/s
                  = 40 Mbps

Peak bandwidth = 40 × 2 = 80 Mbps
```

### Total Bandwidth

| Direction       | Bandwidth | Notes                    |
| --------------- | --------- | ------------------------ |
| Redis incoming  | 160 Mbps  | Rate limit operations    |
| Redis outgoing  | 80 Mbps   | Responses                |
| **Total**       | **~240 Mbps** | Peak requirement         |

---

## Memory Estimation

### Redis Memory Requirements

**Per-Node Memory:**
```
Active keys per node: 10M / 6 nodes = 1.67M keys/node
Memory per key: ~1.7 KB (average, including overhead)
Memory per node = 1.67M × 1.7 KB
                = 2.84 GB

With 50% overhead (Redis internal structures):
Memory per node = 2.84 GB × 1.5
                = 4.26 GB

With 2x buffer for growth:
Memory per node = 4.26 GB × 2
                = 8.5 GB
```

### Total Memory Requirements

| Component              | Memory    | Notes                      |
| ---------------------- | --------- | -------------------------- |
| Per Redis node (×6)    | 51 GB     | 8.5 GB × 6 nodes           |
| Application memory     | 16 GB     | Per application server     |
| **Total**              | **~67 GB**| Distributed across cluster |

---

## Server Estimation

### Rate Limiter Service

**Calculation:**
```
Rate limit checks: 100K QPS (average)
Checks per server: 10,000/second (limited by Redis latency)

Servers needed = 100K / 10K = 10 servers
With redundancy: 15 servers
```

**Rate Limiter Server Specs:**
```
CPU: 8 cores (lightweight operations)
Memory: 16 GB
Network: 10 Gbps
Disk: 100 GB SSD
```

### Redis Cluster

**Calculation:**
```
Operations: 100K QPS (average)
Operations per node: 20,000/second

Nodes needed = 100K / 20K = 5 nodes
With redundancy: 6 nodes
```

**Redis Node Specs:**
```
CPU: 8 cores
Memory: 64 GB (for counters and windows)
Network: 10 Gbps
Disk: 500 GB SSD (AOF persistence)
```

### Server Summary

| Component           | Servers | Specs                           |
| ------------------- | ------- | ------------------------------- |
| Rate limiter service| 15      | 8 cores, 16 GB RAM, 100 GB SSD |
| Redis cluster       | 6       | 8 cores, 64 GB RAM, 500 GB SSD |
| **Total**          | **21**  |                                 |

---

## Cost Estimation

### Compute Costs (AWS)

| Component           | Instance Type | Count | Monthly Cost |
| ------------------- | ------------- | ----- | ------------ |
| Rate limiter service| c6i.2xlarge   | 15    | $3,600       |
| Redis cluster       | r6i.2xlarge   | 6     | $2,160       |
| **Compute Total**   |               |       | **$5,760**   |

### Storage Costs

| Type                | Size/Requests | Monthly Cost |
| ------------------- | ------------- | ------------ |
| EBS (SSD)           | 3.6 TB        | $360         |
| **Storage Total**   |               | **$360**     |

### Network Costs

```
Data transfer: 240 Mbps peak = minimal
Cross-AZ: ~$200/month

Network total: ~$200/month
```

### Total Monthly Cost

| Category    | Cost         |
| ----------- | ------------ |
| Compute     | $5,760      |
| Storage     | $360        |
| Network     | $200        |
| Misc (20%)  | $1,264      |
| **Total**   | **~$7,584/month** |

### Cost per Request

```
Monthly requests: 100K QPS × 86,400 × 30 = 259.2 billion requests/month
Monthly cost: $7,584

Cost per request = $7,584 / 259.2B
                 = $0.000000029
                 = $0.029 per million requests
```

---

## Scaling Projections

### 10x Scale (1M QPS)

| Metric          | Current     | 10x Scale   |
| --------------- | ----------- | ----------- |
| QPS             | 100K        | 1M          |
| Servers         | 21          | 210         |
| Storage         | 20 GB       | 200 GB      |
| Monthly cost    | $7.6K       | $76K        |

### Bottlenecks at Scale

1. **Redis network latency**: More nodes = more network hops
   - Solution: Regional deployment, local caching

2. **Key distribution**: Hot keys cause uneven load
   - Solution: Key sharding, local caching

3. **Memory requirements**: More active keys = more memory
   - Solution: Aggressive TTL, better eviction

---

## Quick Reference Numbers

### Interview Mental Math

```
100,000 QPS
20 GB storage
21 servers
$7.6K/month
$0.029 per million requests
```

### Key Ratios

```
Rate limit checks per server: 10K/second
Memory per Redis node: 8.5 GB
Latency p95: < 10ms
Hit rate: N/A (all requests checked)
```

---

## Summary

| Aspect              | Value                                  |
| ------------------- | -------------------------------------- |
| QPS                 | 100,000                                |
| Storage             | 20 GB                                  |
| Servers             | ~21                                    |
| Monthly cost        | ~$7,584                                |
| Cost per request    | $0.029 per million requests            |
