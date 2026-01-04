# Pastebin - Capacity Estimation

## Traffic Estimation

### Given Assumptions

| Metric | Value | Source |
|--------|-------|--------|
| New pastes per month | 10 million | Business requirement |
| Read:Write ratio | 10:1 | Typical for content sharing |
| Paste views per month | 100 million | 10M × 10 |

### QPS Calculations

**Paste Creation (Writes):**
```
10 million pastes / month
= 10M / 30 days
= 333,333 pastes / day
= 333,333 / 86,400 seconds
≈ 3.9 pastes/second (average)

Peak QPS (assume 5x average):
= 3.9 × 5
≈ 20 pastes/second (peak)
```

**Paste Views (Reads):**
```
100 million views / month
= 100M / 30 days
= 3.33 million views / day
= 3.33M / 86,400 seconds
≈ 39 views/second (average)

Peak QPS (assume 5x average):
= 39 × 5
≈ 200 views/second (peak)
```

**Math Verification:**
- Assumptions: 100M views/month, 30 days/month, 86,400 seconds/day, 5x peak multiplier
- Average QPS: 100,000,000 / 30 / 86,400 = 38.58 QPS ≈ 39 QPS
- Peak QPS: 38.58 × 5 = 192.9 QPS ≈ 200 QPS (rounded)
- **DOC MATCHES:** Traffic summary table shows 40 avg, 200 peak for paste views ✅

### Traffic Summary

| Operation | Average QPS | Peak QPS (5x) |
|-----------|-------------|---------------|
| Paste Creation | 4 | 20 |
| Paste View | 40 | 200 |
| **Total** | **44** | **220** |

**Observation**: This is much lower traffic than URL Shortener. The challenge here is **storage**, not throughput.

---

## Storage Estimation

### Paste Size Distribution

Based on real-world usage patterns:

| Size Category | Percentage | Average Size | Description |
|---------------|------------|--------------|-------------|
| Tiny | 30% | 500 bytes | Single function, config snippet |
| Small | 40% | 5 KB | Small script, log excerpt |
| Medium | 20% | 50 KB | Full file, longer log |
| Large | 9% | 500 KB | Multiple files, large log |
| Very Large | 1% | 5 MB | Huge dumps, data exports |

**Weighted Average:**
```
= (0.30 × 500) + (0.40 × 5,000) + (0.20 × 50,000) + (0.09 × 500,000) + (0.01 × 5,000,000)
= 150 + 2,000 + 10,000 + 45,000 + 50,000
= 107,150 bytes
≈ 100 KB average (rounded for easy math)
```

**Math Verification:**
- Assumptions: Size distribution (30% @ 500B, 40% @ 5KB, 20% @ 50KB, 9% @ 500KB, 1% @ 5MB)
- Calculation: (0.30×500) + (0.40×5,000) + (0.20×50,000) + (0.09×500,000) + (0.01×5,000,000) = 107,150 bytes
- Conversion: 107,150 bytes ÷ 1,024 = 104.6 KB ≈ 100 KB (rounded)
- **DOC MATCHES:** Storage calculations use 100 KB average ✅

**Note**: The 1% of very large pastes contribute significantly to storage.

### Storage Per Paste

| Component | Size | Notes |
|-----------|------|-------|
| Content | 100 KB (avg) | Actual paste text |
| Metadata | 500 bytes | ID, timestamps, settings |
| Indexes | 200 bytes | Database overhead |
| **Total** | ~100 KB | Per paste |

### Monthly Storage Growth

```
New pastes per month = 10 million
Storage per paste = 100 KB

Monthly storage = 10M × 100 KB = 1 TB / month
```

**Math Verification:**
- Assumptions: 10M pastes/month, 100 KB per paste
- Calculation: 10,000,000 × 100 KB = 1,000,000,000 KB = 1,000,000 MB = 1,000 GB = 1 TB
- **DOC MATCHES:** Annual projection table shows 1 TB/month, 12 TB/year ✅

### Annual Storage Projection

| Timeframe | Raw Storage | With Compression (50%) | With Replication (3x) |
|-----------|-------------|------------------------|----------------------|
| 1 month | 1 TB | 500 GB | 1.5 TB |
| 6 months | 6 TB | 3 TB | 9 TB |
| 1 year | 12 TB | 6 TB | 18 TB |
| 5 years | 60 TB | 30 TB | 90 TB |

**Key Insight**: Compression is crucial. Text compresses very well (50-80% reduction).

### Storage After Expiration

With 30-day default expiration:
```
Active pastes at any time ≈ 10M (one month's worth)
Storage for active pastes ≈ 1 TB

Permanent pastes (10% of users choose "never expire"):
= 10M × 0.10 × 12 months = 12M pastes/year
= 12M × 100 KB = 1.2 TB/year of permanent storage
```

---

## Bandwidth Estimation

### Incoming Bandwidth (Uploads)

```
Paste creation:
- Average paste size: 100 KB
- Peak QPS: 20

Incoming bandwidth = 20 × 100 KB = 2 MB/s = 16 Mbps
```

**Math Verification:**
- Assumptions: 20 paste creations/sec (peak), 100 KB average paste size
- Calculation: 20 ops/sec × 100 KB = 2,000 KB/sec = 2 MB/sec
- Conversion: 2 MB/sec × 8 bits/byte = 16 Mbps
- **DOC MATCHES:** Bandwidth summary table shows 16 Mbps incoming ✅

### Outgoing Bandwidth (Downloads)

```
Paste views:
- Average paste size: 100 KB
- Peak QPS: 200

Outgoing bandwidth = 200 × 100 KB = 20 MB/s = 160 Mbps
```

**Math Verification:**
- Assumptions: 200 paste views/sec (peak), 100 KB average paste size
- Calculation: 200 ops/sec × 100 KB = 20,000 KB/sec = 20 MB/sec
- Conversion: 20 MB/sec × 8 bits/byte = 160 Mbps
- **DOC MATCHES:** Bandwidth summary table shows 160 Mbps outgoing (without CDN) ✅

### With CDN Caching

Popular pastes are cached at CDN edges:
```
Assume 70% cache hit rate
Origin bandwidth = 160 Mbps × 0.30 = 48 Mbps
```

**Math Verification:**
- Assumptions: 160 Mbps outgoing bandwidth, 70% CDN cache hit rate (30% miss rate)
- Calculation: 160 Mbps × 0.30 = 48 Mbps origin bandwidth
- **DOC MATCHES:** Bandwidth summary table shows 48 Mbps outgoing with CDN ✅

### Bandwidth Summary

| Direction | Without CDN | With CDN (70% hit) |
|-----------|-------------|-------------------|
| Incoming | 16 Mbps | 16 Mbps (no caching for writes) |
| Outgoing | 160 Mbps | 48 Mbps |
| **Total** | 176 Mbps | 64 Mbps |

---

## Memory Estimation (Cache Sizing)

### What to Cache?

1. **Metadata**: Paste info (small, cache all active)
2. **Content**: Full paste content (large, cache popular only)

### Metadata Cache

```
Active pastes: 10 million
Metadata per paste: 500 bytes

Metadata cache = 10M × 500 bytes = 5 GB
```

### Content Cache

Using 80/20 rule: 20% of pastes get 80% of views

```
Hot pastes (20%): 2 million
Average size: 100 KB

Content cache = 2M × 100 KB = 200 GB
```

**Problem**: 200 GB is too much for Redis.

### Tiered Caching Strategy

| Tier | What | Size | Storage |
|------|------|------|---------|
| L1: Local (Caffeine) | Top 10K pastes | 10K × 100KB | 1 GB per server |
| L2: Redis | Top 100K pastes | 100K × 100KB | 10 GB |
| L3: CDN | All accessed pastes | Variable | CDN handles |
| L4: Object Storage | All pastes | All | S3/GCS |

### Cache Hit Rate Targets

| Tier | Expected Hit Rate |
|------|-------------------|
| L1 (Local) | 20% |
| L2 (Redis) | 40% |
| L3 (CDN) | 30% |
| L4 (Storage) | 10% |

---

## Server Estimation

### Application Servers

```
Peak QPS: 220
Requests per server: 100/second (conservative for content-heavy)

Servers needed = 220 / 100 = 2.2 servers
With 2x headroom = 5 servers
```

### Storage Servers

Using object storage (S3/GCS):
- Managed service, no server estimation needed
- Pay per GB stored and transferred

### Database Servers (Metadata)

```
Metadata queries: 220 QPS peak
PostgreSQL can handle 5,000+ QPS

Servers needed: 1 primary + 2 replicas for HA
```

### Cache Servers

```
Redis for metadata and hot content:
- 10 GB for hot content
- 5 GB for metadata

Total: 15 GB → 1 Redis cluster (3 nodes, 8GB each)
```

### Server Summary

| Component | Count | Specs |
|-----------|-------|-------|
| Load Balancer | 2 | HA pair |
| Application Servers | 5 | 4 CPU, 8 GB RAM |
| Cache (Redis) | 3 | 8 GB RAM each |
| Database (PostgreSQL) | 3 | 4 CPU, 16 GB RAM |
| Object Storage | N/A | Managed (S3/GCS) |
| **Total Servers** | 13 | |

---

## Cost Estimation

### Monthly Costs (AWS)

| Component | Specs | Monthly Cost |
|-----------|-------|--------------|
| EC2 (App Servers) | 5 × m5.xlarge | $750 |
| EC2 (Database) | 3 × r5.large | $450 |
| ElastiCache (Redis) | 3 × r5.large | $450 |
| S3 Storage | 6 TB (6 months active) | $140 |
| S3 Transfer | 50 TB/month | $4,500 |
| CloudFront CDN | 100 TB/month | $8,500 |
| **Total** | | **~$15,000/month** |

### Cost Breakdown

```
Storage: $140 (1%)
Compute: $1,650 (11%)
Data Transfer: $13,000 (88%)
```

**Key Insight**: Data transfer is the dominant cost. CDN optimization is critical.

### Cost Optimization Strategies

1. **Compression**: Reduce transfer by 50-80%
2. **CDN caching**: Reduce origin egress
3. **Regional storage**: Store in cheapest region
4. **Lifecycle policies**: Delete expired pastes promptly
5. **Reserved instances**: 30-40% savings on compute

### Cost Per Paste

```
Monthly cost: $15,000
Monthly pastes: 10 million
Monthly views: 100 million

Cost per paste created: $0.0015 ($1.50 per 1000)
Cost per view: $0.00015 ($0.15 per 1000)
```

---

## Comparison with URL Shortener

| Metric | URL Shortener | Pastebin |
|--------|---------------|----------|
| Peak QPS | 12,000 | 220 |
| Storage/item | 500 bytes | 100 KB |
| Total storage (1 year) | 3 TB | 12 TB |
| Bandwidth (peak) | 100 Mbps | 176 Mbps |
| Primary bottleneck | Throughput | Storage/Bandwidth |
| Main cost driver | Compute | Data Transfer |

---

## Growth Projections

### Year-over-Year (30% growth)

| Year | Pastes/Month | Storage (cumulative) | Monthly Cost |
|------|--------------|---------------------|--------------|
| Year 1 | 10M | 12 TB | $15,000 |
| Year 2 | 13M | 28 TB | $22,000 |
| Year 3 | 17M | 48 TB | $32,000 |
| Year 4 | 22M | 74 TB | $45,000 |
| Year 5 | 29M | 109 TB | $62,000 |

### Scaling Triggers

| Metric | Threshold | Action |
|--------|-----------|--------|
| Storage usage | > 80% capacity | Add storage, review retention |
| CDN cache hit rate | < 60% | Increase cache TTL |
| App server CPU | > 70% | Scale horizontally |
| Database connections | > 80% | Add read replicas |
| Monthly cost | > budget | Review compression, retention |

---

## Summary

| Metric | Value |
|--------|-------|
| **Traffic** | |
| Paste creation QPS (peak) | 20 |
| Paste view QPS (peak) | 200 |
| **Storage** | |
| Average paste size | 100 KB |
| Monthly storage growth | 1 TB |
| Annual storage | 12 TB |
| **Bandwidth** | |
| Peak incoming | 16 Mbps |
| Peak outgoing | 160 Mbps |
| **Memory** | |
| Metadata cache | 5 GB |
| Content cache (hot) | 10 GB |
| **Infrastructure** | |
| Application servers | 5 |
| Database servers | 3 |
| Cache servers | 3 |
| **Cost** | |
| Monthly infrastructure | ~$15,000 |
| Cost per paste | ~$0.0015 |
| Main cost driver | Data transfer (88%) |

