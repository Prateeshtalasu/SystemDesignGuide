# File Storage System - Capacity Estimation

## 1. User and Traffic Estimates

### User Base

```
Total Registered Users:     500,000,000 (500M)
Monthly Active Users (MAU): 150,000,000 (150M) - 30% of total
Daily Active Users (DAU):    50,000,000 (50M)  - 33% of MAU
Concurrent Users (Peak):      5,000,000 (5M)   - 10% of DAU
```

### User Segmentation

```
┌─────────────────────────────────────────────────────────────────┐
│                    User Segmentation                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Free Users:        400M (80%)  - 15 GB storage limit           │
│  Personal Pro:       75M (15%)  - 2 TB storage limit            │
│  Business:           20M (4%)   - Unlimited (fair use)          │
│  Enterprise:          5M (1%)   - Unlimited with admin          │
│                                                                  │
│  Average Storage Used:                                           │
│  • Free:     3 GB (20% of limit)                                │
│  • Pro:    200 GB (10% of limit)                                │
│  • Business: 50 GB per user                                     │
│  • Enterprise: 100 GB per user                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Daily Operations

```
Per Active User Per Day:
├── File Uploads:      2 files (average)
├── File Downloads:    5 files (average)
├── File Views:       10 files (preview without download)
├── Folder Operations: 3 operations
├── Search Queries:    2 queries
├── Share Operations:  0.5 operations
└── Sync Checks:      100 checks (desktop client polling)

Total Daily Operations:
├── File Uploads:      100M uploads/day
├── File Downloads:    250M downloads/day
├── File Views:        500M views/day
├── Folder Operations: 150M operations/day
├── Search Queries:    100M queries/day
├── Share Operations:   25M shares/day
└── Sync Checks:        5B checks/day
```

**Math Verification:**
- Assumptions: 50M DAU, 2 uploads/user/day, 5 downloads/user/day, 10 views/user/day, 3 folder ops/user/day, 2 searches/user/day, 0.5 shares/user/day, 100 sync checks/user/day
- Uploads: 50,000,000 × 2 = 100,000,000 uploads/day
- Downloads: 50,000,000 × 5 = 250,000,000 downloads/day
- Views: 50,000,000 × 10 = 500,000,000 views/day
- Folder ops: 50,000,000 × 3 = 150,000,000 operations/day
- Searches: 50,000,000 × 2 = 100,000,000 queries/day
- Shares: 50,000,000 × 0.5 = 25,000,000 shares/day
- Sync checks: 50,000,000 × 100 = 5,000,000,000 checks/day
- **DOC MATCHES:** Daily operations calculations verified ✅

---

## 2. Storage Estimation

### Total Storage Calculation

```
Storage by User Type:

Free Users (400M × 3 GB average):
= 1,200 PB (1.2 EB)

Personal Pro (75M × 200 GB average):
= 15,000 PB (15 EB)

Business (20M × 50 GB average):
= 1,000 PB (1 EB)

Enterprise (5M × 100 GB average):
= 500 PB (0.5 EB)

Total User Storage: ~17.7 EB
With Replication (3x): ~53 EB

**Math Verification:**
- Assumptions: 400M free users × 3 GB, 75M pro × 200 GB, 20M business × 50 GB, 5M enterprise × 100 GB, 3x replication
- Free: 400,000,000 × 3 GB = 1,200,000 GB = 1,200 PB = 1.2 EB
- Pro: 75,000,000 × 200 GB = 15,000,000 GB = 15,000 PB = 15 EB
- Business: 20,000,000 × 50 GB = 1,000,000 GB = 1,000 PB = 1 EB
- Enterprise: 5,000,000 × 100 GB = 500,000 GB = 500 PB = 0.5 EB
- Total: 1.2 + 15 + 1 + 0.5 = 17.7 EB
- With replication: 17.7 × 3 = 53.1 EB ≈ 53 EB
- **DOC MATCHES:** Storage calculations verified ✅

Note: This is Dropbox-scale. For interview, often use smaller numbers.
Let's use 500M users × 1 GB average = 500 PB for calculations.
```

### Storage Breakdown

```
┌─────────────────────────────────────────────────────────────────┐
│                    Storage Breakdown                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  File Content Storage (Object Storage):                         │
│  ├── Active Files:        400 PB                                │
│  ├── Version History:      75 PB (180 days retention)           │
│  ├── Trash:                25 PB (30 days retention)            │
│  └── Total:               500 PB                                │
│                                                                  │
│  Metadata Storage (Database):                                    │
│  ├── File Metadata:        10 TB (100B files × 100 bytes)       │
│  ├── User Data:             1 TB (500M users × 2 KB)            │
│  ├── Sharing Data:          2 TB                                │
│  ├── Activity Logs:        50 TB (90 days)                      │
│  └── Total:                63 TB                                │
│                                                                  │
│  Index Storage:                                                  │
│  ├── Search Index:         50 TB (10% of text content)          │
│  ├── File Hash Index:       5 TB (deduplication)                │
│  └── Total:                55 TB                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Storage Growth

```
Daily Storage Growth:
├── New Uploads:     100M files × 5 MB average = 500 TB/day
├── Deletions:      -200 TB/day (estimated)
├── Net Growth:      300 TB/day
└── Annual Growth:   ~110 PB/year

Version History:
├── Files Modified:  50M files/day
├── Version Size:    5 MB average
├── Daily Versions:  250 TB/day
├── Retention:       180 days
└── Total Versions:  45 PB (steady state)
```

---

## 3. Traffic Estimation

### Request Rates

```
Peak Traffic Calculation:
- DAU: 50M users
- Peak hours: 20% of traffic in 2 hours
- Peak multiplier: 3x average

Average Requests Per Second:
├── Uploads:     100M / 86,400 = 1,157/sec
├── Downloads:   250M / 86,400 = 2,894/sec
├── Views:       500M / 86,400 = 5,787/sec
├── Metadata:    150M / 86,400 = 1,736/sec
├── Search:      100M / 86,400 = 1,157/sec
├── Sync:        5B / 86,400   = 57,870/sec
└── Total:       ~71,000/sec average
```

**Math Verification:**
- Assumptions: 100M uploads/day, 250M downloads/day, 500M views/day, 150M metadata ops/day, 100M searches/day, 5B sync checks/day, 86,400 seconds/day
- Uploads: 100,000,000 / 86,400 = 1,157.41 QPS ≈ 1,157 QPS
- Downloads: 250,000,000 / 86,400 = 2,893.52 QPS ≈ 2,894 QPS
- Views: 500,000,000 / 86,400 = 5,787.04 QPS ≈ 5,787 QPS
- Metadata: 150,000,000 / 86,400 = 1,736.11 QPS ≈ 1,736 QPS
- Search: 100,000,000 / 86,400 = 1,157.41 QPS ≈ 1,157 QPS
- Sync: 5,000,000,000 / 86,400 = 57,870.37 QPS ≈ 57,870 QPS
- Total: 1,157 + 2,894 + 5,787 + 1,736 + 1,157 + 57,870 = 70,601 QPS ≈ 71,000 QPS (rounded)
- **DOC MATCHES:** Request rate calculations verified ✅

Peak Requests Per Second (3x):
├── Uploads:      3,500/sec
├── Downloads:    8,700/sec
├── Views:       17,400/sec
├── Metadata:     5,200/sec
├── Search:       3,500/sec
├── Sync:       175,000/sec
└── Total:      ~213,000/sec peak
```

### Bandwidth Estimation

```
Upload Bandwidth:
├── Files: 100M × 5 MB = 500 TB/day
├── Average: 500 TB / 86,400 = 5.8 GB/sec
├── Peak (3x): 17.4 GB/sec = 139 Gbps
```

**Math Verification:**
- Assumptions: 100M uploads/day, 5 MB average file size, 86,400 seconds/day, 3x peak multiplier
- Daily: 100,000,000 × 5 MB = 500,000,000 MB = 500 TB
- Average: 500 TB / 86,400 = 0.005787 TB/sec = 5.787 GB/sec ≈ 5.8 GB/sec
- Peak: 5.8 × 3 = 17.4 GB/sec
- Peak (Gbps): 17.4 GB/sec × 8 bits/byte = 139.2 Gbps ≈ 139 Gbps
- **DOC MATCHES:** Upload bandwidth calculations verified ✅

Download Bandwidth:
├── Files: 250M × 5 MB = 1.25 PB/day
├── Average: 1.25 PB / 86,400 = 14.5 GB/sec
├── Peak (3x): 43.5 GB/sec = 348 Gbps
```

**Math Verification:**
- Assumptions: 250M downloads/day, 5 MB average file size, 86,400 seconds/day, 3x peak multiplier
- Daily: 250,000,000 × 5 MB = 1,250,000,000 MB = 1,250,000 GB = 1.25 PB
- Average: 1.25 PB / 86,400 = 0.014467 PB/sec = 14.467 GB/sec ≈ 14.5 GB/sec
- Peak: 14.5 × 3 = 43.5 GB/sec
- Peak (Gbps): 43.5 GB/sec × 8 bits/byte = 348 Gbps
- **DOC MATCHES:** Download bandwidth calculations verified ✅

Total Bandwidth:
├── Average: 20.3 GB/sec = 162 Gbps
├── Peak: 61 GB/sec = 487 Gbps
└── Monthly: ~53 PB egress
```

---

## 4. Compute Estimation

### API Servers

```
API Server Sizing:
├── Requests/server: 5,000/sec (metadata operations)
├── Peak requests: 200,000/sec (excluding sync)
├── Servers needed: 200,000 / 5,000 = 40 servers
├── With redundancy (2x): 80 servers
└── Instance type: 16 vCPU, 32 GB RAM

Sync Servers:
├── Sync checks/server: 10,000/sec
├── Peak sync: 175,000/sec
├── Servers needed: 18 servers
├── With redundancy (2x): 36 servers
└── Instance type: 8 vCPU, 16 GB RAM
```

### Upload/Download Workers

```
Upload Workers:
├── Concurrent uploads: 50,000 (at peak)
├── Uploads/worker: 100 concurrent
├── Workers needed: 500
├── Instance type: 8 vCPU, 16 GB RAM, high network

Download Workers:
├── Concurrent downloads: 100,000 (at peak)
├── Downloads/worker: 200 concurrent
├── Workers needed: 500
├── Instance type: 8 vCPU, 16 GB RAM, high network
```

### Background Processing

```
File Processing Workers:
├── Thumbnail generation: 100 workers
├── Preview generation: 50 workers
├── Virus scanning: 100 workers
├── Content indexing: 50 workers
├── Deduplication: 20 workers
└── Total: 320 workers

Instance type: 4 vCPU, 8 GB RAM
```

---

## 5. Database Sizing

### Metadata Database (PostgreSQL)

```
File Metadata Table:
├── Total files: 100 billion
├── Row size: 500 bytes (path, hash, size, timestamps, etc.)
├── Total size: 50 TB
├── With indexes: 75 TB
├── Shards: 100 (750 GB per shard)
└── Replicas: 3 per shard = 300 instances

User Table:
├── Total users: 500 million
├── Row size: 2 KB
├── Total size: 1 TB
├── Shards: 10
└── Replicas: 3 per shard = 30 instances

Sharing Table:
├── Total shares: 5 billion
├── Row size: 200 bytes
├── Total size: 1 TB
├── Shards: 10
└── Replicas: 3 per shard = 30 instances
```

### Activity Log Database (Time-series)

```
Activity Events:
├── Events/day: 500 million
├── Event size: 500 bytes
├── Daily data: 250 GB
├── Retention: 90 days
├── Total size: 22.5 TB
└── Database: TimescaleDB or ClickHouse
```

### Cache Layer (Redis)

```
Redis Cluster:
├── Hot metadata: 10 TB
├── User sessions: 500 GB
├── Rate limiting: 100 GB
├── Sync state: 1 TB
├── Total: ~12 TB
├── Nodes: 200 (64 GB each)
└── Replication: 3x
```

---

## 6. Object Storage

### Storage Tiers

```
┌─────────────────────────────────────────────────────────────────┐
│                    Storage Tier Strategy                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Hot Storage (S3 Standard):                                      │
│  ├── Recently accessed files (< 30 days)                        │
│  ├── Size: 100 PB                                               │
│  ├── Cost: $0.023/GB/month = $2.3M/month                       │
│  └── Access: < 100ms                                            │
│                                                                  │
│  Warm Storage (S3 Infrequent Access):                           │
│  ├── Files accessed 30-180 days ago                             │
│  ├── Size: 200 PB                                               │
│  ├── Cost: $0.0125/GB/month = $2.5M/month                      │
│  └── Access: < 200ms                                            │
│                                                                  │
│  Cold Storage (S3 Glacier):                                      │
│  ├── Files not accessed > 180 days                              │
│  ├── Size: 200 PB                                               │
│  ├── Cost: $0.004/GB/month = $0.8M/month                       │
│  └── Access: 1-5 hours (restore needed)                         │
│                                                                  │
│  Total Storage Cost: ~$5.6M/month                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Replication Strategy

```
Durability through Replication:
├── Within region: 3 copies (AZ redundancy)
├── Cross region: 2 additional copies (DR)
├── Total copies: 5 (for critical data)
├── Durability: 99.999999999% (11 nines)
└── Storage multiplier: 3x (average)

Effective Storage:
├── Logical data: 500 PB
├── Physical storage: 1.5 EB (3x replication)
└── Deduplication savings: ~20%
```

---

## 7. CDN and Edge

### CDN Configuration

```
CDN (CloudFront/Akamai):
├── Edge locations: 200+ globally
├── Cache hit ratio: 60% (for downloads)
├── Bandwidth served from edge: 200 Gbps
├── Origin bandwidth: 80 Gbps
└── Monthly egress: 30 PB

Edge Computing:
├── Thumbnail serving
├── Preview generation (small files)
├── Authentication validation
└── Rate limiting
```

### Regional Distribution

```
Traffic by Region:
├── North America: 35%
├── Europe: 30%
├── Asia Pacific: 25%
├── Rest of World: 10%

Data Centers:
├── US East (Primary)
├── US West
├── EU West (Ireland)
├── EU Central (Frankfurt)
├── Asia (Singapore)
├── Asia (Tokyo)
└── Total: 6 regions
```

---

## 8. Message Queue Sizing

### Kafka Cluster

```
Event Types and Volume:
├── File events: 200M/day (upload, modify, delete)
├── Sync events: 5B/day
├── Share events: 25M/day
├── Activity events: 500M/day
└── Total: ~6B events/day

Kafka Sizing:
├── Events/sec (peak): 200,000
├── Event size: 1 KB average
├── Throughput: 200 MB/sec
├── Retention: 7 days
├── Storage: 200 MB/sec × 86,400 × 7 = 120 TB
├── Partitions: 500 (across topics)
├── Brokers: 30 (with replication factor 3)
└── Instance type: 16 vCPU, 64 GB RAM, NVMe SSD
```

---

## 9. Search Infrastructure

### Elasticsearch Cluster

```
Searchable Content:
├── Text files indexed: 10B files
├── Average text size: 10 KB
├── Total text: 100 TB
├── Index size: 50 TB (compressed)

Elasticsearch Sizing:
├── Data nodes: 100 (500 GB each)
├── Master nodes: 3
├── Coordinating nodes: 10
├── Shards: 1000 (50 GB each)
├── Replicas: 2
└── Instance type: 16 vCPU, 64 GB RAM, NVMe SSD

Query Performance:
├── Queries/sec: 3,500 (peak)
├── Latency p50: 100ms
├── Latency p99: 500ms
└── Index lag: < 5 minutes
```

---

## 10. Summary Tables

### Infrastructure Summary

| Component | Quantity | Specs | Purpose |
|-----------|----------|-------|---------|
| API Servers | 80 | 16 vCPU, 32 GB | Request handling |
| Sync Servers | 36 | 8 vCPU, 16 GB | Sync coordination |
| Upload Workers | 500 | 8 vCPU, 16 GB | File uploads |
| Download Workers | 500 | 8 vCPU, 16 GB | File downloads |
| Processing Workers | 320 | 4 vCPU, 8 GB | Background jobs |
| PostgreSQL Shards | 360 | 16 vCPU, 128 GB | Metadata |
| Redis Nodes | 200 | 64 GB | Caching |
| Kafka Brokers | 30 | 16 vCPU, 64 GB | Messaging |
| Elasticsearch Nodes | 113 | 16 vCPU, 64 GB | Search |

### Storage Summary

| Type | Size | Cost/Month |
|------|------|------------|
| Hot Object Storage | 100 PB | $2.3M |
| Warm Object Storage | 200 PB | $2.5M |
| Cold Object Storage | 200 PB | $0.8M |
| Database Storage | 100 TB | $50K |
| Cache Storage | 12 TB | $30K |
| Search Index | 50 TB | $100K |
| **Total** | **500 PB** | **~$5.8M** |

### Traffic Summary

| Metric | Average | Peak |
|--------|---------|------|
| API Requests/sec | 71,000 | 213,000 |
| Upload Bandwidth | 5.8 GB/s | 17.4 GB/s |
| Download Bandwidth | 14.5 GB/s | 43.5 GB/s |
| Monthly Egress | 53 PB | - |

### Cost Summary (Monthly)

| Category | Cost |
|----------|------|
| Object Storage | $5.6M |
| Compute (EC2) | $2.0M |
| Database | $0.5M |
| CDN/Bandwidth | $3.0M |
| Search | $0.3M |
| Messaging | $0.2M |
| Other (Monitoring, etc.) | $0.4M |
| **Total** | **~$12M/month** |

Cost per user: $12M / 50M DAU = **$0.24/user/month**

