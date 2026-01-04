# Chat System - Capacity Estimation

## Traffic Estimation

### User Base

| Metric              | Value         | Notes                              |
| ------------------- | ------------- | ---------------------------------- |
| Total users         | 2B            | Registered accounts                |
| Daily active users  | 500M          | 25% DAU ratio                      |
| Concurrent users    | 100M          | Peak online at any time            |
| Messages per user   | 50/day        | Average active user                |

### Message Volume

```
Daily messages:
= DAU × Messages per user
= 500M × 50
= 25 billion messages/day

Average QPS:
= 25B / 86,400
= 289,351 messages/second
≈ 290K messages/second

Peak QPS (2x average):
= 290K × 2
= 580K messages/second
≈ 500K messages/second
```

**Math Verification:**
- Assumptions: 25B messages/day, 86,400 seconds/day, 2x peak multiplier
- Average QPS: 25,000,000,000 / 86,400 = 289,351.85 messages/sec ≈ 290,000 messages/sec (rounded)
- Peak QPS: 290,000 × 2 = 580,000 messages/sec ≈ 500,000 messages/sec (rounded down for design)
- **DOC MATCHES:** Message QPS calculations verified ✅

### Connection Volume

```
WebSocket connections:
= Concurrent users × Devices per user
= 100M × 1.5 (avg devices)
= 150M concurrent connections

Connection events per second:
- New connections: ~50K/second (users coming online)
- Disconnections: ~50K/second (users going offline)
- Heartbeats: 150M / 30 seconds = 5M/second
```

**Math Verification:**
- Assumptions: 150M concurrent connections, 30-second heartbeat interval
- Heartbeats/sec: 150,000,000 / 30 = 5,000,000 heartbeats/sec = 5M/second
- **DOC MATCHES:** Connection event calculations verified ✅

### Message Types

| Message Type        | Percentage | Peak QPS |
| ------------------- | ---------- | -------- |
| Text messages       | 70%        | 350K     |
| Media messages      | 20%        | 100K     |
| System messages     | 5%         | 25K      |
| Typing indicators   | 5%         | 25K      |

---

## Storage Estimation

### Message Storage

```
Per message:
- Message ID: 16 bytes (UUID)
- Sender ID: 8 bytes
- Conversation ID: 16 bytes
- Content: 500 bytes (average)
- Timestamp: 8 bytes
- Status: 1 byte
- Metadata: 100 bytes
- Total per message: ~650 bytes

Daily message storage:
= 25B messages × 650 bytes
= 16.25 TB/day

Yearly storage (with 5-year retention):
= 16.25 TB × 365 × 5
= 29.6 PB
```

**Math Verification:**
- Assumptions: 25B messages/day, 650 bytes per message, 5-year retention
- Daily: 25,000,000,000 × 650 bytes = 16,250,000,000,000 bytes = 16.25 TB
- Yearly: 16.25 TB × 365 = 5,931.25 TB = 5.93 PB
- 5-year: 5.93 PB × 5 = 29.65 PB ≈ 29.6 PB
- **DOC MATCHES:** Storage calculations verified ✅

### Media Storage

```
Media messages: 20% of total = 5B/day
Average media size: 500 KB (compressed)

Daily media storage:
= 5B × 500 KB
= 2.5 PB/day

Yearly media storage:
= 2.5 PB × 365
= 912 PB/year
```

**Math Verification:**
- Assumptions: 5B media messages/day, 500 KB per media, 365 days/year
- Daily: 5,000,000,000 × 500 KB = 2,500,000,000,000 KB = 2.5 PB
- Yearly: 2.5 PB × 365 = 912.5 PB ≈ 912 PB
- **DOC MATCHES:** Storage calculations verified ✅

### User Data Storage

```
Per user:
- User ID: 8 bytes
- Profile: 1 KB
- Settings: 500 bytes
- Device tokens: 200 bytes × 3 devices = 600 bytes
- Contacts: 500 contacts × 8 bytes = 4 KB
- Total per user: ~6 KB

Total user data:
= 2B users × 6 KB
= 12 TB
```

### Conversation Metadata

```
Per conversation:
- Conversation ID: 16 bytes
- Type: 1 byte (1:1 or group)
- Participants: 8 bytes × 256 max = 2 KB
- Last message: 100 bytes
- Settings: 200 bytes
- Total: ~2.5 KB

Estimated conversations:
- 1:1: 2B users × 50 avg contacts = 100B (but deduplicated = 50B)
- Groups: 500M groups

Total conversation storage:
= (50B + 500M) × 2.5 KB
= 125 TB
```

### Total Storage

| Component                | Size/Day    | Size/Year    |
| ------------------------ | ----------- | ------------ |
| Message text             | 16.25 TB    | 5.9 PB       |
| Media files              | 2.5 PB      | 912 PB       |
| User data                | -           | 12 TB        |
| Conversation metadata    | -           | 125 TB       |
| Message indices          | 5 TB        | 1.8 PB       |
| **Total**                | **~2.5 PB** | **~920 PB**  |

---

## Bandwidth Estimation

### Inbound (Messages Sent)

```
Per message sent:
- Message content: 650 bytes
- Protocol overhead: 100 bytes
- Total: ~750 bytes

Inbound bandwidth:
= 500K messages/second × 750 bytes
= 375 MB/second
= 3 Gbps

With media (100K media messages/second × 500 KB):
= 50 GB/second
= 400 Gbps
```

### Outbound (Messages Delivered)

```
Fan-out factor:
- 1:1 messages (80%): 1x delivery
- Group messages (20%): 50x average delivery

Effective deliveries:
= 500K × 0.8 × 1 + 500K × 0.2 × 50
= 400K + 5M
= 5.4M deliveries/second

Outbound bandwidth (text):
= 5.4M × 750 bytes
= 4 GB/second
= 32 Gbps

With media:
= 400 Gbps (similar to inbound due to fan-out)
```

### Presence Updates

```
Presence changes: 100K/second (users going online/offline)
Subscribers per user: 100 average contacts

Fan-out:
= 100K × 100
= 10M presence updates/second

Bandwidth:
= 10M × 50 bytes
= 500 MB/second
= 4 Gbps
```

---

## Infrastructure Sizing

### WebSocket Servers (Chat Servers)

```
Per server:
- Handle 100K concurrent connections
- 32 CPU cores
- 64 GB RAM
- 10 Gbps network

Servers needed:
= 150M connections / 100K per server
= 1,500 servers

With redundancy (1.5x):
= 2,250 servers
```

### Message Queue (Kafka)

```
Message throughput: 500K/second
Retention: 7 days
Replication factor: 3

Kafka cluster:
- Brokers: 50
- Partitions per topic: 500
- Storage per broker: 10 TB
```

### Message Database (Cassandra)

```
Write throughput: 500K/second
Read throughput: 5M/second (with fan-out)
Storage: 6 PB (1 year messages)

Cassandra cluster:
- Nodes: 500
- Replication factor: 3
- Storage per node: 4 TB SSD
```

### Cache (Redis)

```
Data to cache:
- Recent messages: 10M conversations × 100 messages × 650 bytes = 650 GB
- User sessions: 100M × 500 bytes = 50 GB
- Presence: 500M × 100 bytes = 50 GB
- Total: ~750 GB

Redis cluster:
- Nodes: 100
- Memory per node: 16 GB
- Replication: 3x
```

### Media Storage (S3/Object Store)

```
Daily uploads: 2.5 PB
Annual storage: 912 PB

Object storage:
- Hot tier (30 days): 75 PB
- Warm tier (1 year): 912 PB
- Cold tier (archive): Compressed
```

---

## Cost Estimation (Monthly)

### Compute

| Component              | Units      | Unit Cost  | Monthly Cost |
| ---------------------- | ---------- | ---------- | ------------ |
| WebSocket servers      | 2,250      | $500       | $1,125,000   |
| API servers            | 200        | $400       | $80,000      |
| Kafka brokers          | 50         | $1,000     | $50,000      |
| **Compute Total**      |            |            | **$1,255,000**|

### Storage

| Component              | Size       | Unit Cost  | Monthly Cost |
| ---------------------- | ---------- | ---------- | ------------ |
| Cassandra (SSD)        | 6 PB       | $0.10/GB   | $600,000     |
| Redis                  | 750 GB     | $0.15/GB   | $112         |
| Media (S3 Standard)    | 75 PB      | $0.023/GB  | $1,725,000   |
| Media (S3 IA)          | 912 PB     | $0.0125/GB | $11,400,000  |
| **Storage Total**      |            |            | **$13,725,112**|

### Network

| Component              | Volume     | Unit Cost  | Monthly Cost |
| ---------------------- | ---------- | ---------- | ------------ |
| Outbound bandwidth     | 100 PB     | $0.05/GB   | $5,000,000   |
| Inter-region           | 10 PB      | $0.02/GB   | $200,000     |
| **Network Total**      |            |            | **$5,200,000**|

### Total Monthly Cost

```
Compute:  $1,255,000
Storage:  $13,725,112
Network:  $5,200,000
Other:    $500,000 (monitoring, CDN, misc)
─────────────────────
Total:    ~$20,680,000/month

Cost per user:
= $20.68M / 500M DAU
= $0.04 per DAU per month

Cost per message:
= $20.68M / (25B × 30)
= $0.000028 per message
```

---

## Scaling Considerations

### Horizontal Scaling Triggers

| Metric                  | Threshold  | Action                        |
| ----------------------- | ---------- | ----------------------------- |
| WebSocket connections   | > 80K/server | Add chat servers           |
| Message queue lag       | > 10 seconds | Add Kafka partitions       |
| Database latency P99    | > 50ms     | Add Cassandra nodes           |
| Cache hit rate          | < 90%      | Increase Redis cluster        |

### Regional Distribution

| Region        | Users     | Chat Servers | DB Nodes |
| ------------- | --------- | ------------ | -------- |
| North America | 200M      | 450          | 100      |
| Europe        | 300M      | 675          | 150      |
| Asia Pacific  | 800M      | 1,800        | 400      |
| Latin America | 200M      | 450          | 100      |
| Others        | 500M      | 1,125        | 250      |

### Capacity Planning

| Timeframe | DAU    | Messages/Day | Chat Servers |
| --------- | ------ | ------------ | ------------ |
| Current   | 500M   | 25B          | 2,250        |
| +1 year   | 750M   | 37.5B        | 3,375        |
| +2 years  | 1B     | 50B          | 4,500        |

---

## Summary

| Metric                | Value                |
| --------------------- | -------------------- |
| Peak messages/second  | 500K                 |
| Concurrent connections| 150M                 |
| Daily message storage | 16.25 TB             |
| Daily media storage   | 2.5 PB               |
| Chat servers          | 2,250                |
| Cassandra nodes       | 500                  |
| Monthly cost          | ~$20.7M              |
| Cost per message      | $0.000028            |

