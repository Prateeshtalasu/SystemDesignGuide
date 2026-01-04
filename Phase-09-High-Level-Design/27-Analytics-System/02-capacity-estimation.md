# Analytics System - Capacity Estimation

## Overview

This document calculates the infrastructure requirements for an analytics system handling 1 million events per second peak, with support for real-time and batch analytics.

---

## Traffic Estimation

### Event Ingestion Rate

**Given:**
- Peak: 1 million events/second
- Average: 500K events/second (50% of peak)
- Operating 24/7

**Calculations:**

```
Peak events/second: 1M
Average events/second: 500K

Daily events = 500K × 86,400 seconds
            = 43.2 billion events/day

Monthly events = 43.2B × 30 days
              = 1.296 trillion events/month

Annual events = 1.296T × 12 months
              = 15.552 trillion events/year
```

**Math Verification:**
- Assumptions: 500K events/sec average, 86,400 seconds/day, 30 days/month, 12 months/year
- Daily: 500,000 × 86,400 = 43,200,000,000 events/day = 43.2 billion events/day
- Monthly: 43,200,000,000 × 30 = 1,296,000,000,000 events/month = 1.296 trillion events/month
- Annual: 1,296,000,000,000 × 12 = 15,552,000,000,000 events/year = 15.552 trillion events/year
- **DOC MATCHES:** Event volume calculations verified ✅
```

**Peak Traffic Considerations:**
- Black Friday, product launches: 3x normal traffic
- Peak capacity needed: 3M events/second
- Geographic distribution: 60% US, 20% EU, 20% Asia

### Read vs Write Ratio

**Write Operations:**
- Event ingestion: 1M writes/second (peak)
- Pre-aggregation writes: 10K writes/second (aggregated metrics)

**Read Operations:**
- Real-time dashboard queries: 1K queries/second
- Batch report queries: 100 queries/second
- Ad-hoc analyst queries: 50 queries/second

**Read/Write Ratio:**
```
Total reads: 1K + 100 + 50 = 1,150 queries/second
Total writes: 1M + 10K = 1,010,000 writes/second

Read/Write ratio: 1,150 / 1,010,000 ≈ 0.1% (write-heavy system)
```

---

## Storage Estimation

### Raw Event Storage

**Event Size Calculation:**
```
Average event (JSON):
  - Event ID: 16 bytes (UUID)
  - User ID: 8 bytes
  - Event type: 20 bytes
  - Timestamp: 8 bytes
  - Properties (JSON): 400 bytes average
  - Metadata (source, IP, etc.): 50 bytes
  Total: ~500 bytes per event
```

**Daily Storage:**
```
Events per day: 43.2 billion
Storage per event: 500 bytes

Daily storage = 43.2B × 500 bytes
             = 21.6 TB/day (uncompressed)

With compression (gzip, 3:1 ratio):
Daily storage = 21.6 TB / 3
             = 7.2 TB/day (compressed)

**Math Verification:**
- Assumptions: 43.2B events/day, 500 bytes per event, 3:1 compression ratio
- Uncompressed: 43,200,000,000 × 500 bytes = 21,600,000,000,000 bytes = 21.6 TB
- Compressed: 21.6 TB / 3 = 7.2 TB
- **DOC MATCHES:** Storage calculations verified ✅
```

**Monthly Storage:**
```
Monthly storage = 7.2 TB × 30 days
               = 216 TB/month (compressed)
```

**Annual Storage:**
```
Annual storage = 216 TB × 12 months
               = 2.592 PB/year (compressed)

With 2-year retention:
Total raw storage = 2.592 PB × 2 years
                  = 5.184 PB
```

**Storage Overhead:**
- Replication (3x): 5.184 PB × 3 = 15.552 PB
- Indexes (20% overhead): 15.552 PB × 1.2 = 18.66 PB
- **Total raw storage: ~19 PB**

### Aggregated Data Storage

**Pre-Aggregated Metrics:**
```
Metrics calculated:
  - Daily aggregates: 1,000 metric types
  - Hourly aggregates: 500 metric types
  - Real-time aggregates: 100 metric types

Daily aggregates:
  - Metrics: 1,000 types
  - Dimensions: 10 dimensions per metric (user segment, country, etc.)
  - Storage per metric: 1 KB
  Daily: 1,000 × 1 KB = 1 MB/day (negligible)

Hourly aggregates:
  - Metrics: 500 types
  - Storage per metric: 100 bytes
  Daily: 500 × 24 hours × 100 bytes = 1.2 MB/day

Real-time aggregates (5-minute windows):
  - Metrics: 100 types
  - Storage per metric: 50 bytes
  Daily: 100 × 288 windows × 50 bytes = 1.44 MB/day
```

**Aggregated Storage:**
```
Daily aggregates: ~3 MB/day
Monthly: 3 MB × 30 = 90 MB/month
Annual: 90 MB × 12 = 1.08 GB/year

With 5-year retention:
Total aggregated: 1.08 GB × 5 = 5.4 GB (negligible compared to raw)
```

### Data Warehouse Storage

**Columnar Storage (Parquet):**
```
Compression ratio: 10:1 (better than gzip for analytics)
Raw data: 21.6 TB/day
Compressed: 21.6 TB / 10 = 2.16 TB/day

Monthly: 2.16 TB × 30 = 64.8 TB/month
Annual: 64.8 TB × 12 = 777.6 TB/year

With 2-year retention:
Total warehouse storage = 777.6 TB × 2 = 1.555 PB
```

**Total Storage Summary:**

| Component | Size | Retention | Notes |
|-----------|------|-----------|-------|
| Raw events (compressed) | 7.2 TB/day | 2 years | Object storage |
| Data warehouse (Parquet) | 2.16 TB/day | 2 years | Columnar format |
| Aggregated metrics | 3 MB/day | 5 years | Pre-computed |
| **Total active storage** | **~1.6 PB** | | Before replication |
| **With replication (3x)** | **~4.8 PB** | | |

---

## Bandwidth Estimation

### Incoming Bandwidth (Event Ingestion)

**Peak Incoming:**
```
Events/second: 1M
Event size: 500 bytes
Peak bandwidth = 1M × 500 bytes
              = 500 MB/second
              = 4 Gbps
```

**Average Incoming:**
```
Average bandwidth = 4 Gbps × 0.5
                 = 2 Gbps
```

**Geographic Distribution:**
- US: 2.4 Gbps (60%)
- EU: 0.8 Gbps (20%)
- Asia: 0.8 Gbps (20%)

### Outgoing Bandwidth (Query Results)

**Query Patterns:**
```
Real-time queries: 1K/second
  - Average result size: 10 KB
  - Bandwidth: 1K × 10 KB = 10 MB/second = 80 Mbps

Batch queries: 100/second
  - Average result size: 1 MB
  - Bandwidth: 100 × 1 MB = 100 MB/second = 800 Mbps

Ad-hoc queries: 50/second
  - Average result size: 5 MB
  - Bandwidth: 50 × 5 MB = 250 MB/second = 2 Gbps
```

**Total Outgoing:**
```
Total outgoing = 80 + 800 + 2000 Mbps
              = 2.88 Gbps (peak)
```

### Internal Bandwidth (Data Pipeline)

**ETL Pipeline:**
```
Events processed: 43.2B/day
Processing window: 24 hours
Throughput: 43.2B / 86,400 = 500K events/second

Data moved: 7.2 TB/day
Bandwidth needed: 7.2 TB / 86,400 seconds
                = 85 MB/second
                = 680 Mbps
```

**Total Bandwidth Summary:**

| Type | Bandwidth | Notes |
|------|-----------|-------|
| Incoming (peak) | 4 Gbps | Event ingestion |
| Incoming (average) | 2 Gbps | Event ingestion |
| Outgoing (peak) | 2.88 Gbps | Query results |
| Internal (ETL) | 680 Mbps | Data pipeline |
| **Total peak** | **~7.5 Gbps** | |

---

## Memory Requirements

### Real-Time Processing

**In-Memory Aggregations:**
```
Time windows:
  - 1-minute windows: 100 metrics × 60 windows = 6,000 aggregates
  - 5-minute windows: 100 metrics × 12 windows = 1,200 aggregates
  - 1-hour windows: 100 metrics × 1 window = 100 aggregates

Memory per aggregate: 1 KB
Total: (6,000 + 1,200 + 100) × 1 KB = 7.3 MB

With 10x buffer for processing: 73 MB
```

**Event Buffering:**
```
Buffer for 5 seconds of events:
Events: 1M/second × 5 seconds = 5M events
Memory: 5M × 500 bytes = 2.5 GB

With replication across nodes: 2.5 GB per node
```

### Cache Requirements

**Query Result Cache:**
```
Common queries: 1,000 query patterns
Average result size: 100 KB
Cache size: 1,000 × 100 KB = 100 MB

With 10x for variations: 1 GB
```

**Metadata Cache:**
```
Event schemas: 10,000 schemas
Schema size: 5 KB each
Cache: 10,000 × 5 KB = 50 MB
```

**Total Memory Summary:**

| Component | Memory | Notes |
|-----------|--------|-------|
| Real-time aggregations | 73 MB | In-memory metrics |
| Event buffer (5 sec) | 2.5 GB | Per processing node |
| Query cache | 1 GB | Result caching |
| Metadata cache | 50 MB | Schema cache |
| **Per node** | **~3.6 GB** | Processing node |
| **Total (10 nodes)** | **~36 GB** | Distributed |

---

## Compute Requirements

### Event Ingestion Servers

**Per Server Capacity:**
```
Events/second per server: 50K (with overhead)
Peak events: 3M/second
Servers needed: 3M / 50K = 60 servers

With 2x redundancy: 120 servers
```

### Stream Processing

**Processing Capacity:**
```
Events/second: 1M
Processing per event: 1 ms
Throughput per core: 1,000 events/second

Cores needed: 1M / 1,000 = 1,000 cores

With 16-core machines:
Machines: 1,000 / 16 = 63 machines

With 2x redundancy: 126 machines
```

### Batch Processing

**Daily Batch Jobs:**
```
Data processed: 7.2 TB/day
Processing window: 4 hours (overnight)
Throughput needed: 7.2 TB / 14,400 seconds = 500 MB/second

With Spark cluster:
  - 10 nodes, 16 cores each = 160 cores
  - Processing: 500 MB/second (achievable)
```

**Compute Summary:**

| Component | Servers | Cores | Notes |
|-----------|---------|-------|-------|
| Ingestion | 120 | 1,920 | API servers |
| Stream processing | 126 | 2,016 | Real-time aggregation |
| Batch processing | 10 | 160 | ETL jobs |
| Query servers | 20 | 320 | Analytics queries |
| **Total** | **276** | **4,416** | |

---

## Growth Projections

### 1 Year Growth

**Assumptions:**
- 50% annual growth in events
- Storage grows proportionally

**Projections:**
```
Events: 1M/second × 1.5 = 1.5M/second
Storage: 19 PB × 1.5 = 28.5 PB
Bandwidth: 4 Gbps × 1.5 = 6 Gbps
```

### 5 Year Growth

**Assumptions:**
- 50% annual growth (compounded)

**Projections:**
```
Events: 1M × (1.5)^5 = 7.6M/second
Storage: 19 PB × 7.6 = 144 PB
Bandwidth: 4 Gbps × 7.6 = 30 Gbps
```

---

## Cost Estimation (Rough)

### Storage Costs

**Object Storage (S3-like):**
```
19 PB × $0.023/GB/month = $437M/month (unrealistic)

With tiered storage:
  - Hot (30 days): 216 TB × $0.023 = $4,968/month
  - Warm (1 year): 2.16 PB × $0.012 = $25,920/month
  - Cold (2+ years): 17 PB × $0.004 = $68,000/month

Total: ~$99K/month
```

**Data Warehouse:**
```
1.6 PB × $5/TB/month = $8,000/month
```

### Compute Costs

**Ingestion Servers:**
```
120 servers × $200/month = $24,000/month
```

**Stream Processing:**
```
126 machines × $500/month = $63,000/month
```

**Total Monthly Cost: ~$194K/month**

---

## Key Takeaways

1. **Write-Heavy System**: 1M writes/second vs 1K reads/second
2. **Massive Storage**: 19 PB with 2-year retention
3. **High Bandwidth**: 4 Gbps incoming, 3 Gbps outgoing
4. **Distributed Compute**: 276 servers, 4,416 cores
5. **Cost Optimization**: Tiered storage, columnar formats critical

---

## FAANG Reference Numbers

For comparison with real-world systems:

- **Google Analytics**: Processes 10+ billion events/day
- **Facebook Analytics**: Handles 100+ billion events/day
- **Amazon Kinesis**: Processes millions of events/second
- **Snowflake**: Stores petabytes, queries in seconds

Our system (1M events/second) is comparable to mid-scale analytics platforms.


