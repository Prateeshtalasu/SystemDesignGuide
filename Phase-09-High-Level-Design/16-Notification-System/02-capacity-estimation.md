# Notification System - Capacity Estimation

## 1. User and Traffic Estimates

### User Base

```
Total Registered Users:     1,000,000,000 (1B)
Daily Active Users (DAU):     200,000,000 (200M)
Devices per User:                       2 (average)
Total Device Tokens:        2,000,000,000 (2B)
```

### Notification Volume

```
Daily Notifications: 10,000,000,000 (10B)
├── Push Notifications:  6B (60%)
├── Email:              2.5B (25%)
├── In-App:              1B (10%)
└── SMS:               500M (5%)

Per Second (Average):
├── Total: 10B / 86,400 = 115,740/sec
├── Push: 69,444/sec
├── Email: 28,935/sec
├── In-App: 11,574/sec
└── SMS: 5,787/sec

Peak (10x average):
├── Total: 1,157,400/sec (~1.2M/sec)
├── Push: 694,440/sec
├── Email: 289,350/sec
├── In-App: 115,740/sec
└── SMS: 57,870/sec
```

**Math Verification:**
- Assumptions: 10B notifications/day, 86,400 seconds/day, 60% push, 25% email, 10% in-app, 5% SMS, 10x peak multiplier
- Total average: 10,000,000,000 / 86,400 = 115,740.74 QPS ≈ 115,740 QPS
- Push average: 6,000,000,000 / 86,400 = 69,444.44 QPS ≈ 69,444 QPS
- Email average: 2,500,000,000 / 86,400 = 28,935.19 QPS ≈ 28,935 QPS
- In-app average: 1,000,000,000 / 86,400 = 11,574.07 QPS ≈ 11,574 QPS
- SMS average: 500,000,000 / 86,400 = 5,787.04 QPS ≈ 5,787 QPS
- Peak total: 115,740 × 10 = 1,157,400 QPS ≈ 1.2M QPS (rounded)
- **DOC MATCHES:** Notification QPS calculations verified ✅
```

### Traffic Patterns

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Daily Traffic Pattern                                 │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Notifications/sec                                                            │
│  1.2M ┤                    ████                                              │
│       │                   ██████                                             │
│  800K ┤                  ████████                                            │
│       │                 ██████████                                           │
│  400K ┤    ████████████████████████████████████                             │
│       │   ██████████████████████████████████████                            │
│  100K ┤████████████████████████████████████████████████                     │
│       └──────────────────────────────────────────────────                   │
│         0    4    8    12   16   20   24  (hours UTC)                       │
│                                                                               │
│  Peak: 9-11 AM, 7-9 PM local times (varies by region)                       │
│  Trough: 2-6 AM local times                                                 │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Storage Estimation

### Device Token Storage

```
Device Tokens:
├── Total tokens: 2B
├── Token size: 200 bytes (including metadata)
├── Total: 2B × 200 bytes = 400 GB
├── With indexes: 600 GB
└── Replication (3x): 1.8 TB

**Math Verification:**
- Assumptions: 2B device tokens, 200 bytes per token, 3x replication
- Base storage: 2,000,000,000 × 200 bytes = 400,000,000,000 bytes = 400 GB
- With indexes: 400 GB × 1.5 = 600 GB (50% index overhead)
- With replication: 600 GB × 3 = 1,800 GB = 1.8 TB
- **DOC MATCHES:** Storage calculations verified ✅

Token Metadata:
├── User ID: 16 bytes (UUID)
├── Device token: 100 bytes
├── Platform: 10 bytes (ios/android/web)
├── App version: 20 bytes
├── Last active: 8 bytes
├── Created at: 8 bytes
├── Preferences: 50 bytes
└── Total: ~200 bytes per token
```

### User Preferences Storage

```
User Preferences:
├── Total users: 1B
├── Preference size: 500 bytes
├── Total: 1B × 500 bytes = 500 GB
├── With indexes: 750 GB
└── Replication (3x): 2.25 TB

**Math Verification:**
- Assumptions: 1B users, 500 bytes per user preference, 3x replication
- Base storage: 1,000,000,000 × 500 bytes = 500,000,000,000 bytes = 500 GB
- With indexes: 500 GB × 1.5 = 750 GB (50% index overhead)
- With replication: 750 GB × 3 = 2,250 GB = 2.25 TB
- **DOC MATCHES:** Storage calculations verified ✅

Preference Fields:
├── Channel preferences (per type): 200 bytes
├── Quiet hours: 50 bytes
├── Timezone: 20 bytes
├── Language: 10 bytes
├── Topic subscriptions: 200 bytes
├── Unsubscribe list: 20 bytes
└── Total: ~500 bytes per user
```

### Notification History

```
Notification Log:
├── Daily notifications: 10B
├── Log entry size: 200 bytes
├── Daily data: 2 TB
├── Retention: 30 days
├── Total: 60 TB
└── With compression: 20 TB

Log Entry:
├── Notification ID: 16 bytes
├── User ID: 16 bytes
├── Channel: 10 bytes
├── Template ID: 16 bytes
├── Status: 10 bytes
├── Timestamps: 24 bytes
├── Metadata: 100 bytes
└── Total: ~200 bytes
```

### Template Storage

```
Templates:
├── Total templates: 100,000
├── Template size: 10 KB (average)
├── Total: 100K × 10 KB = 1 GB
├── Versions (10 per template): 10 GB
└── With replication: 30 GB
```

### Total Storage Summary

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Storage Summary                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Component              │ Raw Size  │ With Replication                       │
│  ───────────────────────┼───────────┼──────────────────                       │
│  Device Tokens          │ 600 GB    │ 1.8 TB                                  │
│  User Preferences       │ 750 GB    │ 2.25 TB                                 │
│  Notification Log       │ 20 TB     │ 60 TB                                   │
│  Templates              │ 10 GB     │ 30 GB                                   │
│  Analytics/Metrics      │ 5 TB      │ 15 TB                                   │
│  ───────────────────────┼───────────┼──────────────────                       │
│  Total                  │ ~27 TB    │ ~80 TB                                  │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Bandwidth Estimation

### Inbound (API Requests)

```
Notification Requests:
├── Requests/sec: 120,000 (average), 1.2M (peak)
├── Request size: 1 KB (average)
├── Bandwidth: 120 MB/sec average, 1.2 GB/sec peak

Webhook Callbacks:
├── Callbacks/sec: 50,000 (delivery status)
├── Callback size: 500 bytes
├── Bandwidth: 25 MB/sec
```

### Outbound (To Providers)

```
Push (APNS/FCM):
├── Notifications/sec: 700,000 (peak)
├── Payload size: 4 KB (with rich content)
├── Bandwidth: 2.8 GB/sec peak

Email (SendGrid):
├── Emails/sec: 290,000 (peak)
├── Email size: 50 KB (average with HTML)
├── Bandwidth: 14.5 GB/sec peak

SMS (Twilio):
├── SMS/sec: 58,000 (peak)
├── SMS size: 200 bytes
├── Bandwidth: 11.6 MB/sec peak

In-App (WebSocket):
├── Messages/sec: 116,000 (peak)
├── Message size: 1 KB
├── Bandwidth: 116 MB/sec peak
```

### Total Bandwidth

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Bandwidth Summary                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Direction    │ Average      │ Peak                                          │
│  ─────────────┼──────────────┼──────────────                                  │
│  Inbound      │ 150 MB/sec   │ 1.5 GB/sec                                    │
│  Outbound     │ 2 GB/sec     │ 18 GB/sec                                     │
│  ─────────────┼──────────────┼──────────────                                  │
│  Total        │ 2.15 GB/sec  │ 19.5 GB/sec                                   │
│               │ = 17 Gbps    │ = 156 Gbps                                    │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Compute Estimation

### API Servers

```
API Gateway:
├── Requests/sec: 1.2M (peak)
├── Requests/server: 20,000
├── Servers needed: 60
├── With redundancy (2x): 120 servers
└── Instance: 8 vCPU, 16 GB RAM

Notification Service:
├── Notifications/sec: 1.2M (peak)
├── Notifications/server: 10,000
├── Servers needed: 120
├── With redundancy (2x): 240 servers
└── Instance: 16 vCPU, 32 GB RAM
```

### Channel Workers

```
Push Workers:
├── Notifications/sec: 700,000 (peak)
├── Per worker: 5,000 (APNS/FCM connection limits)
├── Workers needed: 140
├── With redundancy: 200 workers
└── Instance: 4 vCPU, 8 GB RAM

Email Workers:
├── Emails/sec: 290,000 (peak)
├── Per worker: 2,000
├── Workers needed: 145
├── With redundancy: 200 workers
└── Instance: 4 vCPU, 8 GB RAM

SMS Workers:
├── SMS/sec: 58,000 (peak)
├── Per worker: 1,000
├── Workers needed: 58
├── With redundancy: 100 workers
└── Instance: 2 vCPU, 4 GB RAM

In-App Workers:
├── Messages/sec: 116,000 (peak)
├── Per worker: 5,000
├── Workers needed: 24
├── With redundancy: 50 workers
└── Instance: 4 vCPU, 8 GB RAM
```

### WebSocket Servers (In-App)

```
WebSocket Connections:
├── Concurrent users: 50M (peak)
├── Connections/server: 100,000
├── Servers needed: 500
├── With redundancy: 600 servers
└── Instance: 8 vCPU, 32 GB RAM
```

---

## 5. Database Sizing

### Primary Database (PostgreSQL)

```
Device Tokens:
├── Records: 2B
├── Table size: 400 GB
├── Indexes: 200 GB
├── Shards: 50 (12 GB per shard)
└── Replicas: 3 per shard = 150 instances

User Preferences:
├── Records: 1B
├── Table size: 500 GB
├── Indexes: 250 GB
├── Shards: 25 (30 GB per shard)
└── Replicas: 3 per shard = 75 instances

Templates:
├── Records: 100K
├── Table size: 10 GB
├── Single instance with replicas
└── Replicas: 3 instances
```

### Time-Series Database (Notification Log)

```
ClickHouse/TimescaleDB:
├── Daily ingestion: 2 TB
├── Retention: 30 days
├── Total: 60 TB (20 TB compressed)
├── Nodes: 20 (1 TB each)
└── Replication: 3x
```

### Cache Layer (Redis)

```
Redis Cluster:
├── Device token cache: 100 GB (hot tokens)
├── User preferences: 50 GB
├── Rate limiting: 20 GB
├── Template cache: 5 GB
├── Deduplication: 50 GB
├── Total: 225 GB
├── Nodes: 50 (5 GB each with overhead)
└── Replication: 2x
```

---

## 6. Message Queue Sizing

### Kafka Cluster

```
Topics and Volume:
├── notification-requests: 1.2M/sec peak
├── push-queue: 700K/sec peak
├── email-queue: 290K/sec peak
├── sms-queue: 58K/sec peak
├── inapp-queue: 116K/sec peak
├── delivery-events: 1.2M/sec peak
└── Total: ~4M events/sec peak

Kafka Sizing:
├── Events/sec: 4M (peak)
├── Event size: 1 KB
├── Throughput: 4 GB/sec
├── Retention: 7 days
├── Storage: 4 GB/sec × 86,400 × 7 = 2.4 PB
├── With compression (10x): 240 TB
├── Partitions: 2000 (across topics)
├── Brokers: 50 (with replication factor 3)
└── Instance: 32 vCPU, 128 GB RAM, NVMe SSD
```

---

## 7. Third-Party Provider Capacity

### Push Notification Providers

```
APNS (Apple):
├── Connections: 100 (recommended per app)
├── Throughput: ~10K/sec per connection
├── Total capacity: 1M/sec
├── Our usage: 350K/sec (iOS)

FCM (Google):
├── No connection limit
├── Quota: 240K/min per project (can increase)
├── Our usage: 350K/sec (Android)
├── Multiple projects for scale
```

### Email Providers

```
SendGrid:
├── Tier: Dedicated IP pool
├── Capacity: 1M/hour per IP
├── IPs needed: 30 (for 290K/sec peak burst)
├── Warm-up required

Alternatives:
├── Amazon SES (backup)
├── Mailgun (backup)
└── Multi-provider for redundancy
```

### SMS Providers

```
Twilio:
├── Throughput: 100 SMS/sec per number
├── Numbers needed: 600 (for 58K/sec)
├── Short codes for higher throughput
├── Cost: ~$0.0075/SMS

Alternatives:
├── Vonage (backup)
├── Plivo (backup)
└── Carrier direct for volume
```

---

## 8. Summary Tables

### Infrastructure Summary

| Component | Quantity | Specs | Purpose |
|-----------|----------|-------|---------|
| API Servers | 120 | 8 vCPU, 16 GB | Request handling |
| Notification Service | 240 | 16 vCPU, 32 GB | Processing |
| Push Workers | 200 | 4 vCPU, 8 GB | APNS/FCM delivery |
| Email Workers | 200 | 4 vCPU, 8 GB | Email delivery |
| SMS Workers | 100 | 2 vCPU, 4 GB | SMS delivery |
| In-App Workers | 50 | 4 vCPU, 8 GB | In-app delivery |
| WebSocket Servers | 600 | 8 vCPU, 32 GB | Real-time connections |
| PostgreSQL | 228 | 16 vCPU, 64 GB | Metadata |
| ClickHouse | 20 | 32 vCPU, 128 GB | Analytics |
| Redis | 50 | 32 GB | Caching |
| Kafka | 50 | 32 vCPU, 128 GB | Messaging |

### Cost Summary (Monthly)

| Category | Cost |
|----------|------|
| Compute | $800K |
| Database | $300K |
| Cache | $50K |
| Kafka | $150K |
| Network | $200K |
| Push (Free) | $0 |
| Email (SendGrid) | $250K |
| SMS (Twilio) | $3.75M |
| Monitoring | $50K |
| **Total** | **~$5.5M/month** |

### Cost per Notification

| Channel | Cost per 1000 |
|---------|---------------|
| Push | $0.001 (infra only) |
| Email | $0.10 |
| In-App | $0.001 (infra only) |
| SMS | $7.50 |
| **Blended** | **$0.55** |

Cost per DAU: $5.5M / 200M = **$0.0275/user/month**

