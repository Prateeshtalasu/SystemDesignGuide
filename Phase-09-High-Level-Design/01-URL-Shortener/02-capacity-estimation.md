# URL Shortener - Capacity Estimation (Back-of-Envelope)

## Why Capacity Estimation Matters

Before designing any system, we must understand the scale. A system handling 100 requests/second looks very different from one handling 100,000 requests/second. Capacity estimation helps us:

1. **Choose the right database**: Can a single PostgreSQL handle the load, or do we need sharding?
2. **Size our caches**: How much memory do we need for Redis?
3. **Plan infrastructure**: How many servers? What bandwidth?
4. **Estimate costs**: Will this cost $1,000/month or $100,000/month?

---

## Traffic Estimation

### Given Assumptions

| Metric | Value | Source |
|--------|-------|--------|
| New URLs created per month | 100 million | Business requirement |
| Read:Write ratio | 100:1 | Typical for URL shorteners |
| Redirects per month | 10 billion | 100M × 100 |

### Mental Math Shortcuts

Before diving into calculations, let's establish some useful shortcuts:

```
1 day = 86,400 seconds ≈ 100,000 seconds (for easy math)
1 month = 30 days ≈ 2.5 million seconds
1 year = 365 days ≈ 30 million seconds

1 million requests/day = ~12 QPS
1 billion requests/day = ~12,000 QPS
```

### QPS Calculations

**URL Creation (Writes):**
```
100 million URLs / month
= 100M / 30 days
= 3.33 million URLs / day
= 3.33M / 86,400 seconds
≈ 38.5 URLs/second (average)

Peak QPS (assume 3x average):
= 38.5 × 3
≈ 115 URLs/second (peak)
```

**URL Redirects (Reads):**
```
10 billion redirects / month
= 10B / 30 days
= 333 million redirects / day
= 333M / 86,400 seconds
≈ 3,858 redirects/second (average)

Peak QPS (assume 3x average):
= 3,858 × 3
≈ 11,574 redirects/second (peak)

Round up for safety:
≈ 12,000 redirects/second (peak)
```

### Traffic Summary

| Operation | Average QPS | Peak QPS (3x) |
|-----------|-------------|---------------|
| URL Creation | 40 | 120 |
| URL Redirect | 4,000 | 12,000 |
| **Total** | **4,040** | **12,120** |

### Geographic Distribution

Assuming global service with traffic distribution:
- North America: 35%
- Europe: 30%
- Asia: 25%
- Rest of World: 10%

This affects CDN placement and database replication strategy.

---

## Storage Estimation

### Data Model Size

Let's calculate the size of a single URL record:

| Field | Type | Size | Notes |
|-------|------|------|-------|
| short_code | VARCHAR(8) | 8 bytes | Primary key |
| original_url | VARCHAR(2048) | ~200 bytes avg | Most URLs are 100-300 chars |
| user_id | BIGINT | 8 bytes | Optional, nullable |
| created_at | TIMESTAMP | 8 bytes | Creation time |
| expires_at | TIMESTAMP | 8 bytes | Expiration time |
| click_count | BIGINT | 8 bytes | Analytics counter |
| is_custom | BOOLEAN | 1 byte | Custom alias flag |
| metadata | JSON | ~100 bytes | Additional info |

**Total per record: ~350 bytes**

Let's round up to **500 bytes** per URL to account for:
- Database overhead (indexes, row headers)
- Future fields
- Padding and alignment

### Storage Calculations

**Per Year:**
```
URLs per year = 100M/month × 12 months = 1.2 billion URLs
Storage per year = 1.2B × 500 bytes = 600 GB
```

**5-Year Projection (default retention):**
```
Total URLs = 1.2B × 5 = 6 billion URLs
Total storage = 6B × 500 bytes = 3 TB
```

**With Replication (3x):**
```
Replicated storage = 3 TB × 3 = 9 TB
```

### Analytics Data Storage

Click events are stored separately for analytics:

| Field | Size |
|-------|------|
| short_code | 8 bytes |
| timestamp | 8 bytes |
| ip_hash | 16 bytes |
| referrer | 100 bytes |
| user_agent | 100 bytes |
| country | 2 bytes |
| city | 50 bytes |

**Total per click event: ~300 bytes**

**Analytics Storage:**
```
Clicks per year = 10B/month × 12 = 120 billion clicks
Storage per year = 120B × 300 bytes = 36 TB

5-year analytics = 36 TB × 5 = 180 TB
```

This is significant! We'll need to:
- Aggregate old data (keep daily summaries, discard raw events)
- Use time-series databases optimized for this pattern
- Consider data retention policies

### Storage Summary

| Data Type | 1 Year | 5 Years | Notes |
|-----------|--------|---------|-------|
| URL mappings | 600 GB | 3 TB | Core data |
| URL mappings (replicated) | 1.8 TB | 9 TB | 3x replication |
| Analytics (raw) | 36 TB | 180 TB | Before aggregation |
| Analytics (aggregated) | 1 TB | 5 TB | Daily rollups |
| **Total** | ~4 TB | ~15 TB | Conservative estimate |

---

## Bandwidth Estimation

### Incoming Bandwidth (Requests)

**URL Creation Request:**
```
Request size ≈ 2 KB (headers + long URL)
Peak QPS = 120
Incoming bandwidth = 120 × 2 KB = 240 KB/s ≈ 2 Mbps
```

**Redirect Request:**
```
Request size ≈ 500 bytes (headers + short URL)
Peak QPS = 12,000
Incoming bandwidth = 12,000 × 500 bytes = 6 MB/s ≈ 48 Mbps
```

### Outgoing Bandwidth (Responses)

**URL Creation Response:**
```
Response size ≈ 200 bytes (JSON with short URL)
Peak QPS = 120
Outgoing bandwidth = 120 × 200 bytes = 24 KB/s ≈ 0.2 Mbps
```

**Redirect Response:**
```
Response size ≈ 500 bytes (HTTP 301 + headers)
Peak QPS = 12,000
Outgoing bandwidth = 12,000 × 500 bytes = 6 MB/s ≈ 48 Mbps
```

### Bandwidth Summary

| Direction | Average | Peak |
|-----------|---------|------|
| Incoming | ~20 Mbps | ~50 Mbps |
| Outgoing | ~20 Mbps | ~50 Mbps |
| **Total** | ~40 Mbps | ~100 Mbps |

This is very manageable. Bandwidth is not a bottleneck for this system.

---

## Memory Estimation (Cache Sizing)

### Why Cache?

With a 100:1 read-to-write ratio and 12,000 peak redirect QPS, we need caching. The 80/20 rule suggests 20% of URLs get 80% of traffic.

### Cache Size Calculation

**Hot URLs (frequently accessed):**
```
Total URLs = 6 billion (5-year projection)
Hot URLs (20%) = 1.2 billion URLs
```

That's too many to cache entirely. Let's be more aggressive:

**Daily Active URLs:**
```
Redirects per day = 333 million
Unique URLs accessed per day ≈ 10% of redirects = 33 million
```

**Cache Entry Size:**
```
Key (short_code): 8 bytes
Value (original_url): 200 bytes
Overhead: 50 bytes
Total per entry: ~260 bytes
```

**Cache Memory Required:**
```
33 million URLs × 260 bytes = 8.6 GB
```

With Redis overhead (fragmentation, data structures):
```
Actual memory needed = 8.6 GB × 1.5 = ~13 GB
```

**Recommendation:** Start with 16 GB Redis cache, scale to 32 GB as needed.

### Cache Hit Rate Target

With proper caching:
- Target cache hit rate: 90%+
- Cache miss rate: 10%
- Database QPS = 12,000 × 0.1 = 1,200 QPS

This is manageable for a well-indexed database.

---

## Server Estimation

### Application Servers

**Assumptions:**
- Each server handles 2,000 requests/second
- Peak QPS = 12,000

```
Servers needed = 12,000 / 2,000 = 6 servers
With 50% headroom = 9 servers
Rounded up = 10 application servers
```

### Database Servers

**Read Replicas:**
```
Database can handle ~5,000 QPS per replica
Cache miss QPS = 1,200
Replicas needed = 1 (with headroom)
```

**For high availability:**
- 1 primary (writes)
- 2 read replicas (reads + failover)
- Total: 3 database servers

### Cache Servers

```
Redis can handle 100,000+ operations/second
Peak QPS = 12,000
Servers needed = 1 (with replication for HA)
```

**For high availability:**
- Redis Cluster with 3 nodes (1 primary + 2 replicas)

### Server Summary

| Component | Count | Specs |
|-----------|-------|-------|
| Load Balancer | 2 | HA pair |
| Application Servers | 10 | 4 CPU, 8 GB RAM |
| Cache (Redis) | 3 | 16 GB RAM each |
| Database (Primary) | 1 | 8 CPU, 32 GB RAM, SSD |
| Database (Replicas) | 2 | 8 CPU, 32 GB RAM, SSD |
| **Total** | 18 | |

---

## Common FAANG Interview Numbers

Keep these memorized for quick calculations:

| Metric | Value |
|--------|-------|
| Seconds per day | 86,400 ≈ 100,000 |
| Seconds per month | 2.5 million |
| 1 million requests/day | ~12 QPS |
| 1 billion requests/day | ~12,000 QPS |
| Typical SSD read latency | 0.1 ms |
| Typical HDD read latency | 10 ms |
| Network round trip (same DC) | 0.5 ms |
| Network round trip (cross-region) | 50-100 ms |
| Redis operation | 0.1 ms |
| PostgreSQL query (indexed) | 1-5 ms |

---

## Growth Projections

### Year-over-Year Growth

Assuming 50% YoY growth:

| Year | URLs/Month | Redirects/Month | Storage |
|------|------------|-----------------|---------|
| Year 1 | 100M | 10B | 600 GB |
| Year 2 | 150M | 15B | 1.5 TB |
| Year 3 | 225M | 22.5B | 2.85 TB |
| Year 4 | 337M | 33.7B | 4.9 TB |
| Year 5 | 506M | 50.6B | 8 TB |

### Scaling Triggers

| Metric | Threshold | Action |
|--------|-----------|--------|
| Cache hit rate | < 85% | Increase cache size |
| Database CPU | > 70% | Add read replicas |
| App server CPU | > 70% | Add more servers |
| Storage | > 80% capacity | Provision more storage |
| Redirect latency P99 | > 100ms | Investigate bottleneck |

---

## Cost Estimation (AWS Pricing)

### Monthly Costs (Rough Estimate)

| Component | Specs | Monthly Cost |
|-----------|-------|--------------|
| EC2 (App Servers) | 10 × m5.xlarge | $1,500 |
| EC2 (Database) | 3 × r5.2xlarge | $2,000 |
| ElastiCache (Redis) | 3 × r5.large | $500 |
| RDS Storage | 5 TB SSD | $500 |
| Data Transfer | 100 TB/month | $5,000 |
| Load Balancer | ALB | $200 |
| **Total** | | **~$10,000/month** |

### Cost per Operation

```
Total monthly cost = $10,000
Total monthly operations = 10 billion redirects + 100 million creates
= 10.1 billion operations

Cost per operation = $10,000 / 10.1B = $0.000001
= $1 per million operations
```

This is very cost-effective!

---

## Summary Table

| Metric | Value |
|--------|-------|
| **Traffic** | |
| URL creations/second (peak) | 120 |
| Redirects/second (peak) | 12,000 |
| **Storage** | |
| URL data (5 years) | 3 TB |
| Analytics data (5 years, aggregated) | 5 TB |
| **Memory** | |
| Cache size | 16 GB |
| Target cache hit rate | 90%+ |
| **Bandwidth** | |
| Peak bandwidth | 100 Mbps |
| **Infrastructure** | |
| Application servers | 10 |
| Database servers | 3 |
| Cache servers | 3 |
| **Cost** | |
| Monthly infrastructure | ~$10,000 |
| Cost per million operations | ~$1 |

---

## Interview Tips

### Show Your Work

Always show calculations step by step:
- State assumptions clearly
- Round numbers for easy mental math
- Sanity check your results

### Common Mistakes

1. **Forgetting peak vs average**: Peak is typically 3-5x average
2. **Ignoring replication**: Storage needs multiply with replicas
3. **Underestimating analytics**: Click data is often larger than URL data
4. **Not considering growth**: Design for 5 years, not today

### Good Answers to Pushback

**"Your numbers seem high"**
> "I've built in safety margins. In production, I'd start smaller and scale based on actual metrics. These estimates help us understand the upper bound."

**"How did you get 100:1 read-to-write ratio?"**
> "This is typical for URL shorteners. Each URL is created once but clicked many times. Bitly reports similar ratios. We can adjust if the actual ratio differs."

