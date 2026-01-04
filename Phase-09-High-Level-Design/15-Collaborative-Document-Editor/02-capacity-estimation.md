# Collaborative Document Editor - Capacity Estimation

## 1. User and Traffic Estimates

### User Base

```
Total Registered Users:     1,000,000,000 (1B)
Monthly Active Users (MAU):   500,000,000 (500M) - 50% of total
Daily Active Users (DAU):     200,000,000 (200M) - 40% of MAU
Concurrent Users (Peak):       20,000,000 (20M)  - 10% of DAU
```

### User Segmentation

```
┌─────────────────────────────────────────────────────────────────┐
│                    User Segmentation                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Free Users:        800M (80%)  - 15 GB storage                 │
│  Personal Pro:      150M (15%)  - 100 GB storage                │
│  Business:           40M (4%)   - Unlimited                     │
│  Enterprise:         10M (1%)   - Unlimited + admin             │
│                                                                  │
│  Average Documents per User:                                     │
│  • Free:     5 documents                                        │
│  • Pro:     20 documents                                        │
│  • Business: 50 documents                                       │
│  • Enterprise: 100 documents                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Document Statistics

```
Total Documents: 10 billion
├── Personal documents: 7B (70%)
├── Shared documents: 2B (20%)
└── Team documents: 1B (10%)

Document Size Distribution:
├── < 10 KB:    50% (text only)
├── 10-100 KB:  30% (some formatting)
├── 100 KB-1 MB: 15% (images)
├── 1-10 MB:    4% (many images)
└── > 10 MB:    1% (large documents)

Average Document Size: 100 KB
Average Characters: 50,000
Average Operations per Edit Session: 500
```

---

## 2. Operations Estimation

### Edit Operations

```
Per Active User Per Day:
├── Edit sessions: 3 sessions
├── Operations per session: 500 ops
├── Total operations: 1,500 ops/user/day

Total Daily Operations:
├── DAU: 200M users
├── Operations: 200M × 1,500 = 300 billion ops/day
├── Average: 300B / 86,400 = 3.5 million ops/sec
└── Peak (3x): 10.5 million ops/sec
```

**Math Verification:**
- Assumptions: 200M DAU, 1,500 operations/user/day, 86,400 seconds/day, 3x peak multiplier
- Daily operations: 200,000,000 × 1,500 = 300,000,000,000 operations/day
- Average QPS: 300,000,000,000 / 86,400 = 3,472,222 ops/sec ≈ 3.5 million ops/sec (rounded)
- Peak QPS: 3,500,000 × 3 = 10,500,000 ops/sec = 10.5 million ops/sec
- **DOC MATCHES:** Operations QPS verified ✅

Operation Types:
├── Insert text: 60%
├── Delete text: 25%
├── Format change: 10%
├── Cursor move: 5%
```

### Document Access Patterns

```
Per Day:
├── Document opens: 500M (2.5 per DAU)
├── Document creates: 50M
├── Document shares: 20M
├── Comment adds: 100M
├── Version views: 30M

Requests per Second:
├── Document opens: 500M / 86,400 = 5,787/sec
├── Creates: 50M / 86,400 = 579/sec
├── Shares: 20M / 86,400 = 231/sec
├── Comments: 100M / 86,400 = 1,157/sec
└── Total metadata: ~8,000/sec (peak: 24,000/sec)
```

**Math Verification:**
- Assumptions: 500M opens/day, 50M creates/day, 20M shares/day, 100M comments/day, 86,400 seconds/day, 3x peak multiplier
- Opens: 500,000,000 / 86,400 = 5,787.04 QPS ≈ 5,787 QPS
- Creates: 50,000,000 / 86,400 = 578.7 QPS ≈ 579 QPS
- Shares: 20,000,000 / 86,400 = 231.48 QPS ≈ 231 QPS
- Comments: 100,000,000 / 86,400 = 1,157.41 QPS ≈ 1,157 QPS
- Total: 5,787 + 579 + 231 + 1,157 = 7,754 QPS ≈ 8,000 QPS (rounded)
- Peak: 8,000 × 3 = 24,000 QPS
- **DOC MATCHES:** Metadata request QPS verified ✅
```

### Collaboration Metrics

```
Concurrent Editing Sessions:
├── Total active documents: 50M (at any time)
├── Documents with 1 editor: 45M (90%)
├── Documents with 2-5 editors: 4.5M (9%)
├── Documents with 5-20 editors: 450K (0.9%)
├── Documents with 20+ editors: 50K (0.1%)

WebSocket Connections:
├── Concurrent users: 20M (peak)
├── Connections per user: 1-2 (multiple tabs)
├── Total connections: 30M (peak)
└── Connections per server: 50K
```

---

## 3. Storage Estimation

### Document Storage

```
Document Content:
├── Total documents: 10B
├── Average size: 100 KB
├── Total: 10B × 100 KB = 1 PB

Version History:
├── Average versions per document: 100
├── Delta size: 2 KB average
├── Total: 10B × 100 × 2 KB = 2 PB

Operations Log:
├── Operations per day: 300B
├── Operation size: 100 bytes
├── Daily: 30 TB
├── Retention: 30 days
├── Total: 900 TB

Total Storage: ~4 PB
```

### Storage Breakdown

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Storage Breakdown                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Document Content:                                                            │
│  ├── Current versions:     1 PB                                              │
│  ├── Version history:      2 PB                                              │
│  ├── Media (images):       500 TB                                            │
│  └── Subtotal:             3.5 PB                                            │
│                                                                               │
│  Operations Storage:                                                          │
│  ├── Operations log:       900 TB (30 days)                                  │
│  ├── Real-time buffer:     100 TB                                            │
│  └── Subtotal:             1 PB                                              │
│                                                                               │
│  Metadata:                                                                    │
│  ├── Document metadata:    50 TB                                             │
│  ├── User data:            10 TB                                             │
│  ├── Sharing/permissions:  20 TB                                             │
│  ├── Comments:             100 TB                                            │
│  └── Subtotal:             180 TB                                            │
│                                                                               │
│  Search Index:             200 TB                                             │
│                                                                               │
│  Total:                    ~5 PB                                              │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Bandwidth Estimation

### Real-time Sync Bandwidth

```
Operations Sync:
├── Operations/sec: 10.5M (peak)
├── Operation size: 100 bytes
├── Sync bandwidth: 1.05 GB/sec = 8.4 Gbps

Cursor/Presence Updates:
├── Updates/sec: 5M (peak)
├── Update size: 50 bytes
├── Bandwidth: 250 MB/sec = 2 Gbps

Document Load:
├── Loads/sec: 20,000 (peak)
├── Average doc size: 100 KB
├── Bandwidth: 2 GB/sec = 16 Gbps

Total Real-time Bandwidth: ~27 Gbps (peak)
```

### WebSocket Traffic

```
Per Connection:
├── Heartbeat: 1 message/30 sec = 2 bytes/sec
├── Presence: 1 message/5 sec = 10 bytes/sec
├── Operations: Variable, avg 50 bytes/sec
├── Total: ~62 bytes/sec per connection

Total WebSocket Traffic:
├── Connections: 30M (peak)
├── Bandwidth: 30M × 62 bytes/sec = 1.86 GB/sec
└── Total: 14.9 Gbps
```

---

## 5. Compute Estimation

### API Servers

```
Metadata API:
├── Requests/sec: 24,000 (peak)
├── Requests/server: 2,000
├── Servers needed: 12
├── With redundancy (3x): 36 servers
└── Instance: 8 vCPU, 16 GB RAM

Document API:
├── Loads/sec: 20,000 (peak)
├── Loads/server: 500
├── Servers needed: 40
├── With redundancy (3x): 120 servers
└── Instance: 16 vCPU, 32 GB RAM
```

### WebSocket Servers

```
Connection Handling:
├── Total connections: 30M (peak)
├── Connections/server: 50,000
├── Servers needed: 600
├── With redundancy: 800 servers
└── Instance: 8 vCPU, 32 GB RAM (memory for connections)
```

### OT Processing Servers

```
Operation Transformation:
├── Operations/sec: 10.5M (peak)
├── Ops/server: 50,000
├── Servers needed: 210
├── With redundancy: 300 servers
└── Instance: 16 vCPU, 32 GB RAM
```

### Background Workers

```
Workers:
├── Version snapshots: 50 workers
├── Search indexing: 100 workers
├── Export generation: 50 workers
├── Spell check: 100 workers
├── Cleanup jobs: 20 workers
└── Total: 320 workers

Instance: 4 vCPU, 8 GB RAM
```

---

## 6. Database Sizing

### Document Metadata (PostgreSQL)

```
Documents Table:
├── Total documents: 10B
├── Row size: 500 bytes
├── Total: 5 TB
├── With indexes: 7.5 TB
├── Shards: 100 (75 GB per shard)
└── Replicas: 3 per shard = 300 instances

Users Table:
├── Total users: 1B
├── Row size: 1 KB
├── Total: 1 TB
├── Shards: 20
└── Replicas: 3 per shard = 60 instances

Sharing Table:
├── Total shares: 5B
├── Row size: 200 bytes
├── Total: 1 TB
├── Shards: 20
└── Replicas: 3 per shard = 60 instances
```

### Operations Storage (Time-Series)

```
Operations Log:
├── Daily operations: 300B
├── Operation size: 100 bytes
├── Daily data: 30 TB
├── Retention: 30 days
├── Total: 900 TB
└── Database: Apache Cassandra or ScyllaDB

Partitioning:
├── Partition key: document_id
├── Clustering key: timestamp
├── Nodes: 200 (4.5 TB per node)
└── Replication factor: 3
```

### Real-time State (Redis)

```
Active Document State:
├── Active documents: 50M
├── State size: 10 KB average
├── Total: 500 TB
├── This is too large for Redis alone!

Solution: Tiered caching
├── Hot documents (< 1 hour): Redis (5M docs, 50 TB)
├── Warm documents: Memcached (10M docs)
├── Cold documents: Load from storage

Redis Cluster:
├── Memory: 50 TB
├── Nodes: 800 (64 GB each)
└── Replication: 2x
```

---

## 7. Message Queue Sizing

### Kafka Cluster

```
Event Types:
├── Operation events: 300B/day
├── Presence events: 50B/day
├── Document events: 100M/day
└── Total: ~350B events/day

Kafka Sizing:
├── Events/sec: 4M (average), 12M (peak)
├── Event size: 200 bytes average
├── Throughput: 2.4 GB/sec (peak)
├── Retention: 7 days
├── Storage: 2.4 GB/sec × 86,400 × 7 = 1.4 PB
├── Partitions: 2000 (across topics)
├── Brokers: 100 (with replication factor 3)
└── Instance: 32 vCPU, 128 GB RAM, NVMe SSD
```

### Topic Design

```
Topics:
├── doc-operations: 1000 partitions (by document_id)
├── doc-presence: 500 partitions (by document_id)
├── doc-events: 200 partitions (create, delete, share)
├── user-events: 200 partitions (login, activity)
└── search-index: 100 partitions
```

---

## 8. Search Infrastructure

### Elasticsearch Cluster

```
Searchable Content:
├── Documents indexed: 10B
├── Average text: 50 KB
├── Total text: 500 TB
├── Index size: 250 TB (compressed)

Elasticsearch Sizing:
├── Data nodes: 500 (500 GB each)
├── Master nodes: 5
├── Coordinating nodes: 20
├── Shards: 5000 (50 GB each)
├── Replicas: 2
└── Instance: 32 vCPU, 128 GB RAM, NVMe SSD

Query Performance:
├── Queries/sec: 10,000 (peak)
├── Latency p50: 50ms
├── Latency p99: 200ms
└── Index lag: < 1 minute
```

---

## 9. Summary Tables

### Infrastructure Summary

| Component | Quantity | Specs | Purpose |
|-----------|----------|-------|---------|
| API Servers | 156 | 8-16 vCPU, 16-32 GB | Metadata/Document API |
| WebSocket Servers | 800 | 8 vCPU, 32 GB | Real-time connections |
| OT Servers | 300 | 16 vCPU, 32 GB | Operation transformation |
| Background Workers | 320 | 4 vCPU, 8 GB | Async processing |
| PostgreSQL | 420 | 16 vCPU, 128 GB | Metadata |
| Cassandra | 200 | 16 vCPU, 64 GB | Operations log |
| Redis | 800 | 64 GB | Real-time state |
| Kafka | 100 | 32 vCPU, 128 GB | Event streaming |
| Elasticsearch | 525 | 32 vCPU, 128 GB | Search |

### Storage Summary

| Type | Size | Cost/Month |
|------|------|------------|
| Document Storage | 3.5 PB | $80K |
| Operations Log | 1 PB | $25K |
| Metadata | 180 TB | $5K |
| Search Index | 250 TB | $50K |
| Redis Cache | 50 TB | $200K |
| **Total** | **~5 PB** | **~$360K** |

### Traffic Summary

| Metric | Average | Peak |
|--------|---------|------|
| Operations/sec | 3.5M | 10.5M |
| WebSocket Connections | 10M | 30M |
| Document Loads/sec | 6K | 20K |
| Real-time Bandwidth | 10 Gbps | 30 Gbps |

### Cost Summary (Monthly)

| Category | Cost |
|----------|------|
| Compute | $3.0M |
| Storage | $0.4M |
| Database (managed) | $1.5M |
| Network/CDN | $0.5M |
| Kafka (managed) | $0.3M |
| Search | $0.5M |
| Other | $0.3M |
| **Total** | **~$6.5M/month** |

Cost per DAU: $6.5M / 200M = **$0.033/user/month**

