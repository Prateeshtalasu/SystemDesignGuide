# Distributed Message Queue - Capacity Estimation

## 1. Throughput Estimates

### Message Volume

```
Write Throughput Target: 10,000,000 messages/second
Read Throughput Target: 30,000,000 messages/second (3x write due to multiple consumers)

Message Size Distribution:
├── < 100 bytes:   20% (metrics, heartbeats)
├── 100-1 KB:      50% (events, commands)
├── 1-10 KB:       25% (documents, logs)
├── 10-100 KB:      4% (images, small files)
└── 100 KB-10 MB:   1% (large payloads)

Average Message Size: 1 KB
Peak Message Size: 10 MB
```

### Data Throughput

```
Write Data Throughput:
├── Messages/sec: 10,000,000
├── Avg size: 1 KB
├── Data rate: 10 GB/sec = 80 Gbps

Read Data Throughput:
├── Messages/sec: 30,000,000
├── Avg size: 1 KB
├── Data rate: 30 GB/sec = 240 Gbps

With Replication (RF=3):
├── Internal replication: 20 GB/sec
├── Total write I/O: 30 GB/sec
└── Total cluster throughput: 60 GB/sec
```

**Math Verification:**
- Assumptions: 10M messages/sec write, 30M messages/sec read, 1 KB average message size, RF=3
- Write data: 10,000,000 × 1 KB = 10,000,000 KB/sec = 10 GB/sec
- Write bandwidth: 10 GB/sec × 8 bits/byte = 80 Gbps
- Read data: 30,000,000 × 1 KB = 30,000,000 KB/sec = 30 GB/sec
- Read bandwidth: 30 GB/sec × 8 bits/byte = 240 Gbps
- Internal replication: (RF-1) × write = 2 × 10 GB/sec = 20 GB/sec
- Total write I/O: 10 GB/sec (original) + 20 GB/sec (replication) = 30 GB/sec
- Total cluster: 30 GB/sec (write) + 30 GB/sec (read) = 60 GB/sec
- **DOC MATCHES:** Throughput calculations verified ✅
```

---

## 2. Storage Estimation

### Daily Storage Growth

```
Daily Message Volume:
├── Messages/day: 10M/sec × 86,400 = 864 billion messages
├── Data/day: 10 GB/sec × 86,400 = 864 TB/day

With Replication (RF=3):
├── Raw storage/day: 864 TB × 3 = 2.6 PB/day
├── With compression (50%): 1.3 PB/day
└── Effective storage/day: ~1.3 PB

**Math Verification:**
- Assumptions: 10 GB/sec write rate, 86,400 seconds/day, RF=3, 50% compression
- Daily data: 10 GB/sec × 86,400 sec = 864,000 GB = 864 TB
- With replication: 864 TB × 3 = 2,592 TB = 2.6 PB
- With compression: 2.6 PB × 0.5 = 1.3 PB
- **DOC MATCHES:** Storage calculations verified ✅
```

### Retention-Based Storage

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    Storage by Retention Period                                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Retention    │ Raw Data    │ With RF=3   │ Compressed                       │
│  ─────────────┼─────────────┼─────────────┼────────────                       │
│  1 hour       │ 36 TB       │ 108 TB      │ 54 TB                            │
│  1 day        │ 864 TB      │ 2.6 PB      │ 1.3 PB                           │
│  7 days       │ 6 PB        │ 18 PB       │ 9 PB                             │
│  30 days      │ 26 PB       │ 78 PB       │ 39 PB                            │
│                                                                               │
│  Default (7 days): 9 PB total cluster storage                                │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Storage Per Broker

```
Target: 100 brokers
Storage per broker: 9 PB / 100 = 90 TB per broker

Broker Storage Configuration:
├── Data disks: 12 × 8 TB NVMe SSD = 96 TB
├── Usable (80%): 77 TB
├── Overhead (logs, index): 10 TB
└── Available for data: 67 TB

With growth buffer: Need 120 brokers
```

---

## 3. Partition and Topic Estimates

### Topic Distribution

```
Total Topics: 100,000
├── High-throughput (> 100K msg/sec): 1,000 topics
├── Medium-throughput (1K-100K msg/sec): 10,000 topics
├── Low-throughput (< 1K msg/sec): 89,000 topics

Partitions per Topic:
├── High-throughput: 1000 partitions average
├── Medium-throughput: 100 partitions average
├── Low-throughput: 10 partitions average

Total Partitions:
├── High: 1,000 × 1,000 = 1,000,000
├── Medium: 10,000 × 100 = 1,000,000
├── Low: 89,000 × 10 = 890,000
└── Total: ~3,000,000 partitions

With RF=3: 9,000,000 partition replicas
```

### Partitions Per Broker

```
Brokers: 120
Partition replicas: 9,000,000
Replicas per broker: 75,000

Recommended limit: 4,000 partitions per broker (Kafka)
Our design: More brokers or fewer partitions needed

Revised:
├── Brokers: 300
├── Replicas per broker: 30,000
└── Within recommended limits (with optimization)
```

---

## 4. Broker Sizing

### Single Broker Specifications

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Broker Specifications                                 │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  CPU:                                                                         │
│  ├── Cores: 32 (for network I/O, compression)                               │
│  ├── Type: High-frequency (3.0+ GHz)                                        │
│  └── Utilization target: 60%                                                │
│                                                                               │
│  Memory:                                                                      │
│  ├── Total: 256 GB                                                          │
│  ├── JVM Heap: 32 GB                                                        │
│  ├── Page Cache: 200 GB (critical for performance)                          │
│  └── OS/Other: 24 GB                                                        │
│                                                                               │
│  Storage:                                                                     │
│  ├── Type: NVMe SSD (for IOPS)                                              │
│  ├── Capacity: 12 × 8 TB = 96 TB                                            │
│  ├── IOPS: 500K+ per disk                                                   │
│  └── Throughput: 3 GB/sec per disk                                          │
│                                                                               │
│  Network:                                                                     │
│  ├── Bandwidth: 25 Gbps (minimum)                                           │
│  ├── Recommended: 100 Gbps                                                  │
│  └── Low latency: < 0.5ms intra-DC                                          │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Cluster Sizing

```
Write throughput per broker:
├── Target: 10M msg/sec ÷ 300 brokers = 33K msg/sec per broker
├── Data: 10 GB/sec ÷ 300 = 33 MB/sec per broker (write)
├── With replication: 100 MB/sec per broker (total write I/O)
└── Well within SSD capability

Read throughput per broker:
├── Target: 30M msg/sec ÷ 300 = 100K msg/sec per broker
├── Data: 30 GB/sec ÷ 300 = 100 MB/sec per broker
└── Served from page cache (fast)

Network per broker:
├── Write: 33 MB/sec from producers
├── Replication: 66 MB/sec to followers
├── Read: 100 MB/sec to consumers
├── Total: ~200 MB/sec = 1.6 Gbps
└── Well within 25 Gbps capacity
```

---

## 5. Controller Cluster Sizing

### Controller Requirements

```
Controller Responsibilities:
├── Metadata management (topics, partitions)
├── Broker registration and health
├── Leader election coordination
├── Partition reassignment

Metadata Size:
├── Topics: 100K × 1 KB = 100 MB
├── Partitions: 3M × 200 bytes = 600 MB
├── Consumer groups: 10K × 10 KB = 100 MB
├── Total metadata: ~1 GB

Controller Cluster:
├── Nodes: 5 (for quorum)
├── Type: Raft-based consensus
├── Memory: 32 GB per node
├── Storage: 100 GB SSD per node
└── Network: 10 Gbps
```

---

## 6. Consumer Group Estimates

### Consumer Groups

```
Total Consumer Groups: 50,000
├── Real-time processing: 10,000 groups
├── Analytics/Batch: 20,000 groups
├── Monitoring: 5,000 groups
├── Other: 15,000 groups

Consumers per Group:
├── Small (1-5 consumers): 40,000 groups
├── Medium (5-50 consumers): 8,000 groups
├── Large (50+ consumers): 2,000 groups

Total Consumers: ~500,000 consumer instances
```

### Offset Storage

```
Offset Commits:
├── Consumers: 500,000
├── Commit interval: 5 seconds
├── Commits/sec: 100,000

Offset Storage (__consumer_offsets topic):
├── Partitions: 50
├── Retention: 7 days
├── Size: ~50 GB
```

---

## 7. Network Estimation

### Internal Network (Broker-to-Broker)

```
Replication Traffic:
├── Write rate: 10 GB/sec
├── Replication factor: 3
├── Replication traffic: 20 GB/sec (2 copies)
├── Distributed across 300 brokers
└── Per broker: ~67 MB/sec replication

Total Internal Traffic: 20 GB/sec = 160 Gbps
```

### External Network (Client-to-Broker)

```
Producer Traffic:
├── Write rate: 10 GB/sec
├── Distributed across 300 brokers
└── Per broker: ~33 MB/sec

Consumer Traffic:
├── Read rate: 30 GB/sec
├── Distributed across 300 brokers
└── Per broker: ~100 MB/sec

Total External Traffic: 40 GB/sec = 320 Gbps
```

### Network Topology

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Network Topology                                      │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Rack 1              Rack 2              Rack 3              Rack N          │
│  ┌─────────┐        ┌─────────┐        ┌─────────┐        ┌─────────┐       │
│  │Broker 1 │        │Broker 2 │        │Broker 3 │        │Broker N │       │
│  │Broker 4 │        │Broker 5 │        │Broker 6 │        │  ...    │       │
│  │  ...    │        │  ...    │        │  ...    │        │         │       │
│  └────┬────┘        └────┬────┘        └────┬────┘        └────┬────┘       │
│       │                  │                  │                  │             │
│  ┌────┴────┐        ┌────┴────┐        ┌────┴────┐        ┌────┴────┐       │
│  │ToR Switch│       │ToR Switch│       │ToR Switch│       │ToR Switch│      │
│  │ 100 Gbps │       │ 100 Gbps │       │ 100 Gbps │       │ 100 Gbps │      │
│  └────┬────┘        └────┬────┘        └────┬────┘        └────┬────┘       │
│       │                  │                  │                  │             │
│       └──────────────────┴──────────────────┴──────────────────┘             │
│                              │                                               │
│                    ┌─────────┴─────────┐                                     │
│                    │   Spine Switches   │                                    │
│                    │    400 Gbps        │                                    │
│                    └───────────────────┘                                     │
│                                                                               │
│  Rack-aware replication ensures replicas on different racks                  │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Summary Tables

### Infrastructure Summary

| Component | Quantity | Specs | Purpose |
|-----------|----------|-------|---------|
| Brokers | 300 | 32 vCPU, 256 GB RAM, 96 TB SSD | Message storage |
| Controllers | 5 | 16 vCPU, 32 GB RAM, 100 GB SSD | Metadata |
| ZooKeeper (legacy) | 5 | 8 vCPU, 16 GB RAM | Coordination |
| Load Balancers | 10 | - | Client routing |

### Storage Summary

| Metric | Value |
|--------|-------|
| Total Raw Storage | 28.8 PB (300 × 96 TB) |
| Usable Storage | 20 PB |
| Data with RF=3 | 9 PB (7-day retention) |
| Growth Rate | 1.3 PB/day |

### Throughput Summary

| Metric | Value |
|--------|-------|
| Write Throughput | 10M msg/sec, 10 GB/sec |
| Read Throughput | 30M msg/sec, 30 GB/sec |
| Replication | 20 GB/sec |
| Total I/O | 60 GB/sec |

### Cost Summary (Monthly)

| Category | Cost |
|----------|------|
| Brokers (300 × $3,000) | $900,000 |
| Controllers (5 × $500) | $2,500 |
| Network | $100,000 |
| Storage | $500,000 |
| Operations | $100,000 |
| **Total** | **~$1.6M/month** |

Cost per message: $1.6M / (10M × 86,400 × 30) = **$0.000000006/message**
Cost per GB: $1.6M / (10 GB × 86,400 × 30) = **$0.00006/GB**

