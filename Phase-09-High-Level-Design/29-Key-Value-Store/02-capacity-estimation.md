# Key-Value Store - Capacity Estimation

## Overview

This document calculates infrastructure requirements for a key-value store handling 100M keys, 1M operations/second.

---

## Traffic Estimation

### Operations Per Second

**Given:**
- Total QPS: 1M operations/second
- Read/Write ratio: 80% reads, 20% writes
- Peak multiplier: 2x

**Calculations:**

```
Reads/second = 1M × 0.8 = 800K reads/second
Writes/second = 1M × 0.2 = 200K writes/second

Peak QPS = 1M × 2 = 2M operations/second
```

**Math Verification:**
- Assumptions: 1M operations/sec total, 80% reads, 20% writes, 2x peak multiplier
- Reads: 1,000,000 × 0.8 = 800,000 reads/sec = 800K reads/sec
- Writes: 1,000,000 × 0.2 = 200,000 writes/sec = 200K writes/sec
- Peak: 1,000,000 × 2 = 2,000,000 operations/sec = 2M operations/sec
- **DOC MATCHES:** Operation QPS calculations verified ✅
```

---

## Storage Estimation

### Memory Storage

**Key-Value Sizes:**
```
Average key: 50 bytes
Average value: 1 KB
Overhead: 100 bytes (metadata, pointers)
Total per key-value: ~1.15 KB
```

**Total Memory:**
```
Keys: 100M
Storage per key: 1.15 KB

Total memory = 100M × 1.15 KB
             = 115 GB

With replication (3x):
Total memory = 115 GB × 3
             = 345 GB

**Math Verification:**
- Assumptions: 115 GB per node, 3 nodes
- Calculation: 115 GB × 3 = 345 GB
- **DOC MATCHES:** Storage calculations verified ✅

**Data Structure Distribution:**
```
Strings (60%): 69 GB
Hashes (20%): 23 GB
Sorted Sets (10%): 11.5 GB
Lists (5%): 5.75 GB
Sets (5%): 5.75 GB
```

### Disk Storage (Persistence)

**RDB Snapshots:**
```
Snapshot size: 115 GB (compressed: 50 GB)
Frequency: Daily
Retention: 7 days
Total: 50 GB × 7 = 350 GB
```

**AOF (Append-Only File):**
```
Writes/second: 200K
Average entry: 100 bytes
Daily AOF: 200K × 86,400 × 100 bytes = 1.73 TB/day

With compression (3:1): 577 GB/day
With rewrite: Keep last 24 hours = 577 GB
```

**Total Disk Storage:**
```
RDB: 350 GB
AOF: 577 GB
Total: 927 GB

With 3x replication: 2.78 TB
```

---

## Bandwidth Estimation

### Internal Bandwidth (Replication)

**Replication Traffic:**
```
Writes/second: 200K
Data per write: 1.15 KB
Replication bandwidth = 200K × 1.15 KB
                      = 230 MB/second
                      = 1.84 Gbps
```

### Client Bandwidth

**Read Traffic:**
```
Reads/second: 800K
Average response: 1 KB
Bandwidth = 800K × 1 KB
          = 800 MB/second
          = 6.4 Gbps
```

**Write Traffic:**
```
Writes/second: 200K
Average request: 1.15 KB
Bandwidth = 200K × 1.15 KB
          = 230 MB/second
          = 1.84 Gbps
```

**Total Bandwidth:**
```
Client: 6.4 + 1.84 = 8.24 Gbps
Replication: 1.84 Gbps
Total: ~10 Gbps
```

---

## Memory Requirements

### Per Node Memory

**Data Storage:**
```
100M keys / 10 nodes = 10M keys per node
Memory per node = 10M × 1.15 KB = 11.5 GB
```

**Overhead:**
```
Indexes: 20% = 2.3 GB
Buffer: 10% = 1.15 GB
Total per node: ~15 GB
```

**With Safety Margin:**
```
Per node: 20 GB
Total (10 nodes): 200 GB
```

---

## Compute Requirements

### Per Node Capacity

**Operations per node:**
```
1M ops/second / 10 nodes = 100K ops/second per node
```

**Servers Needed:**
```
10 nodes (data)
3 nodes (replicas)
Total: 13 nodes
```

**With Redundancy:**
```
20 nodes (data + replicas)
```

---

## Growth Projections

### 1 Year Growth

**Assumptions:** 50% growth

**Projections:**
```
Keys: 100M × 1.5 = 150M
QPS: 1M × 1.5 = 1.5M ops/second
Memory: 115 GB × 1.5 = 172.5 GB
```

### 5 Year Growth

**Assumptions:** 50% annual growth

**Projections:**
```
Keys: 100M × (1.5)^5 = 759M
QPS: 1M × (1.5)^5 = 7.6M ops/second
Memory: 115 GB × 7.6 = 874 GB
```

---

## Cost Estimation

### Storage Costs

**Memory:**
```
345 GB × $0.10/GB/month = $34.50/month
```

**Disk:**
```
2.78 TB × $0.05/GB/month = $139/month
```

### Compute Costs

**Servers:**
```
20 nodes × $500/month = $10,000/month
```

**Total: ~$10,200/month**

---

## Key Takeaways

1. **Memory-Intensive**: 345 GB total memory
2. **High Throughput**: 1M ops/second
3. **High Bandwidth**: 10 Gbps
4. **Moderate Compute**: 20 nodes
5. **Cost**: ~$10K/month

---

## FAANG Reference

- **Redis**: Handles millions of ops/second
- **Memcached**: Similar performance characteristics
- **Amazon ElastiCache**: Managed Redis service

Our system (1M ops/second, 100M keys) is comparable to mid-scale key-value stores.

