# Distributed Configuration Management - Capacity Estimation

## Overview

This document calculates infrastructure requirements for a configuration management system handling 10,000 services, 100K configuration keys, with 100K reads/second and 1K writes/second.

---

## Traffic Estimation

### Read Operations

**Given:**
- Services: 10,000
- Reads per service per second: 10 (polling every 100ms)
- Peak multiplier: 2x

**Calculations:**

```
Base reads/second = 10,000 services × 10 reads/second
                 = 100,000 reads/second

Peak reads/second = 100,000 × 2
                  = 200,000 reads/second
```

**Math Verification:**
- Assumptions: 10,000 services, 10 reads/sec per service (polling every 100ms), 2x peak multiplier
- Base: 10,000 × 10 = 100,000 reads/sec
- Peak: 100,000 × 2 = 200,000 reads/sec
- **DOC MATCHES:** Read QPS calculations verified ✅
```

**Read Patterns:**
- Hot configurations (80%): Frequently accessed, cached
- Cold configurations (20%): Rarely accessed, database

### Write Operations

**Given:**
- Configuration updates: 1,000/second peak
- Average: 500/second
- Read/Write ratio: 1000:1

**Calculations:**

```
Peak writes/second: 1,000
Average writes/second: 500

Daily writes = 500 × 86,400 seconds
            = 43.2 million writes/day
```

**Math Verification:**
- Assumptions: 500 writes/sec average, 86,400 seconds/day
- Daily: 500 × 86,400 = 43,200,000 writes/day = 43.2 million writes/day
- **DOC MATCHES:** Write volume calculations verified ✅
```

---

## Storage Estimation

### Configuration Storage

**Configuration Size:**
```
Average configuration (JSON):
  - Key: 50 bytes
  - Value: 1 KB average
  - Metadata: 200 bytes (version, timestamp, etc.)
  Total: ~1.25 KB per configuration
```

**Total Storage:**
```
Configurations: 100,000 keys
Storage per config: 1.25 KB

Total storage = 100,000 × 1.25 KB
             = 125 MB (current)

With versioning (keep 10 versions):
Total storage = 125 MB × 10
             = 1.25 GB

With 3x replication:
Total storage = 1.25 GB × 3
             = 3.75 GB

**Math Verification:**
- Assumptions: 100K configurations, 1.25 KB per config, 10 versions, 3x replication
- Base: 100,000 × 1.25 KB = 125,000 KB = 125 MB
- With versions: 125 MB × 10 = 1,250 MB = 1.25 GB
- With replication: 1.25 GB × 3 = 3.75 GB
- **DOC MATCHES:** Storage calculations verified ✅
```

**Growth Projections:**
```
1 year: 100K × 1.2 = 120K configs = 1.5 GB
5 years: 100K × 1.2^5 = 249K configs = 3.1 GB
```

### Version History Storage

**Version History:**
```
Updates per day: 43.2 million
Average config updated: 1 time/day
Configs with history: 100,000

Versions per config: 10 (retention)
Storage per version: 1.25 KB

Version history = 100,000 × 10 × 1.25 KB
               = 1.25 GB

With 3x replication: 3.75 GB
```

### Metadata Storage

**Metadata:**
```
Per configuration:
  - Version: 8 bytes
  - Timestamp: 8 bytes
  - User: 50 bytes
  - Change log: 200 bytes
  Total: ~266 bytes

Total metadata = 100,000 × 266 bytes
              = 26.6 MB (negligible)
```

**Total Storage Summary:**

| Component | Size | Notes |
|-----------|------|-------|
| Current configs | 125 MB | Active configurations |
| Version history | 1.25 GB | 10 versions per config |
| Metadata | 27 MB | Change tracking |
| **Total** | **~1.4 GB** | Before replication |
| **With replication (3x)** | **~4.2 GB** | |

---

## Bandwidth Estimation

### Incoming Bandwidth (Writes)

**Peak Incoming:**
```
Writes/second: 1,000
Config size: 1.25 KB
Peak bandwidth = 1,000 × 1.25 KB
               = 1.25 MB/second
               = 10 Mbps
```

### Outgoing Bandwidth (Reads)

**Peak Outgoing:**
```
Reads/second: 200,000
Config size: 1.25 KB (but 80% cached, smaller response)
Average response: 0.5 KB (cached responses are smaller)

Peak bandwidth = 200,000 × 0.5 KB
               = 100 MB/second
               = 800 Mbps
```

### Update Propagation Bandwidth

**Push Updates:**
```
Updates/second: 1,000
Services to notify: 10,000 (but only affected services)
Average affected services: 100 per update
Update size: 1.25 KB

Propagation bandwidth = 1,000 × 100 × 1.25 KB
                      = 125 MB/second
                      = 1 Gbps
```

**Total Bandwidth Summary:**

| Type | Bandwidth | Notes |
|------|-----------|-------|
| Incoming (writes) | 10 Mbps | Configuration updates |
| Outgoing (reads) | 800 Mbps | Configuration retrieval |
| Propagation (pushes) | 1 Gbps | Update notifications |
| **Total peak** | **~1.8 Gbps** | |

---

## Memory Requirements

### Cache Requirements

**Hot Configuration Cache:**
```
Hot configs (80%): 80,000 configs
Cache size: 80,000 × 1.25 KB = 100 MB

With overhead (Redis): 100 MB × 1.2 = 120 MB
```

**Query Result Cache:**
```
Common queries: 1,000 query patterns
Average result: 10 KB
Cache size: 1,000 × 10 KB = 10 MB
```

**Update Subscription Cache:**
```
Active subscriptions: 10,000 services
Subscription metadata: 1 KB per service
Cache size: 10,000 × 1 KB = 10 MB
```

**Total Memory Summary:**

| Component | Memory | Notes |
|-----------|--------|-------|
| Hot config cache | 120 MB | Frequently accessed |
| Query cache | 10 MB | Query results |
| Subscription cache | 10 MB | WebSocket connections |
| **Total** | **~140 MB** | Per Redis node |
| **Total (3 nodes)** | **~420 MB** | Redis cluster |

---

## Compute Requirements

### Configuration Service

**Per Server Capacity:**
```
Reads/second per server: 10,000 (with caching)
Peak reads: 200,000/second
Servers needed: 200,000 / 10,000 = 20 servers

With 2x redundancy: 40 servers
```

### Update Propagation Service

**Per Server Capacity:**
```
Updates/second per server: 100
Peak updates: 1,000/second
Servers needed: 1,000 / 100 = 10 servers

With 2x redundancy: 20 servers
```

### WebSocket Service (Push Updates)

**Per Server Capacity:**
```
Connections per server: 1,000
Total connections: 10,000 services
Servers needed: 10,000 / 1,000 = 10 servers

With 2x redundancy: 20 servers
```

**Compute Summary:**

| Component | Servers | Notes |
|-----------|---------|-------|
| Configuration service | 40 | Read/write operations |
| Update propagation | 20 | Push notifications |
| WebSocket service | 20 | Real-time connections |
| **Total** | **80** | |

---

## Growth Projections

### 1 Year Growth

**Assumptions:**
- 20% growth in services and configs

**Projections:**
```
Services: 10,000 × 1.2 = 12,000
Configs: 100,000 × 1.2 = 120,000
Reads: 200,000 × 1.2 = 240,000/second
Writes: 1,000 × 1.2 = 1,200/second
```

### 5 Year Growth

**Assumptions:**
- 20% annual growth (compounded)

**Projections:**
```
Services: 10,000 × (1.2)^5 = 24,883
Configs: 100,000 × (1.2)^5 = 248,832
Reads: 200,000 × (1.2)^5 = 497,664/second
Writes: 1,000 × (1.2)^5 = 2,488/second
```

---

## Cost Estimation (Rough)

### Storage Costs

**PostgreSQL:**
```
4.2 GB × $0.10/GB/month = $0.42/month (negligible)
```

**Redis:**
```
420 MB × $0.05/GB/month = $0.02/month (negligible)
```

### Compute Costs

**Configuration Service:**
```
40 servers × $200/month = $8,000/month
```

**Update Propagation:**
```
20 servers × $150/month = $3,000/month
```

**WebSocket Service:**
```
20 servers × $150/month = $3,000/month
```

**Total Monthly Cost: ~$14K/month**

---

## Key Takeaways

1. **Read-Heavy System**: 1000:1 read/write ratio
2. **Small Storage**: ~4.2 GB total (manageable)
3. **High Bandwidth**: 1.8 Gbps peak (update propagation)
4. **Moderate Compute**: 80 servers
5. **Cost-Effective**: ~$14K/month

---

## FAANG Reference Numbers

For comparison with real-world systems:

- **Consul**: Handles 10K+ services, millions of keys
- **etcd**: Used by Kubernetes, handles 100K+ keys
- **AWS Systems Manager Parameter Store**: Unlimited parameters
- **HashiCorp Vault**: Manages secrets for large enterprises

Our system (10K services, 100K keys) is comparable to mid-scale configuration management platforms.

