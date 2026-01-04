# Instagram / Photo Sharing - Capacity Estimation

## Traffic Estimation

### User Base

| Metric              | Value         | Notes                              |
| ------------------- | ------------- | ---------------------------------- |
| Total users         | 2B            | Registered accounts                |
| Daily active users  | 500M          | 25% DAU ratio                      |
| Monthly active users| 1.5B          | 75% MAU ratio                      |
| Concurrent users    | 50M           | Peak online at any time            |

### Photo Uploads

```
Daily uploads:
= DAU × Upload rate
= 500M × 0.1 (10% of users upload daily)
= 50M photos/day

Upload QPS:
= 50M / 86,400
= 579 uploads/second
≈ 600 uploads/second

Peak QPS (5x):
= 600 × 5
= 3,000 uploads/second
```

**Math Verification:**
- Assumptions: 50M uploads/day, 86,400 seconds/day, 5x peak multiplier
- Average QPS: 50,000,000 / 86,400 = 578.7 uploads/sec ≈ 600 uploads/sec (rounded)
- Peak QPS: 600 × 5 = 3,000 uploads/sec
- **DOC MATCHES:** Upload QPS calculations verified ✅

### Feed Requests

```
Feed loads per user per day: 10

Daily feed requests:
= 500M × 10
= 5 billion requests/day

Average QPS:
= 5B / 86,400
= 57,870 QPS
≈ 60K QPS

Peak QPS (3x):
= 60K × 3
= 180K QPS
```

**Math Verification:**
- Assumptions: 5B feed requests/day, 86,400 seconds/day, 3x peak multiplier
- Average QPS: 5,000,000,000 / 86,400 = 57,870.37 QPS ≈ 60,000 QPS (rounded)
- Peak QPS: 60,000 × 3 = 180,000 QPS
- **DOC MATCHES:** Feed request QPS verified ✅

### Story Views

```
Stories posted per day:
= 500M × 0.2 (20% post stories)
= 100M stories/day

Story views per day:
= 100M stories × 50 avg views
= 5 billion story views/day

Story QPS:
= 5B / 86,400
= 58K QPS
```

**Math Verification:**
- Assumptions: 5B story views/day, 86,400 seconds/day
- Average QPS: 5,000,000,000 / 86,400 = 57,870.37 QPS ≈ 58,000 QPS (rounded)
- **DOC MATCHES:** Story view QPS verified ✅

---

## Storage Estimation

### Photo Storage

```
Daily uploads: 50M photos

Per photo storage:
- Original: 2 MB
- Large (1080px): 300 KB
- Medium (640px): 100 KB
- Thumbnail (150px): 10 KB
- Total per photo: ~2.5 MB

Daily photo storage:
= 50M × 2.5 MB
= 125 TB/day

Yearly storage:
= 125 TB × 365
= 45.6 PB/year

5-year storage:
= 45.6 PB × 5
= 228 PB
```

**Math Verification:**
- Assumptions: 50M photos/day, 2.5 MB per photo, 365 days/year, 5-year retention
- Daily: 50,000,000 × 2.5 MB = 125,000,000 MB = 125 TB
- Yearly: 125 TB × 365 = 45,625 TB = 45.6 PB
- 5-year: 45.6 PB × 5 = 228 PB
- **DOC MATCHES:** Storage calculations verified ✅

### Story Storage

```
Daily stories: 100M
Average story size: 500 KB (photo) or 5 MB (video)
Weighted average: 1 MB

Daily story storage:
= 100M × 1 MB
= 100 TB/day

But stories expire after 24 hours:
Active story storage = 100 TB (constant)
```

**Math Verification:**
- Assumptions: 100M stories/day, 1 MB per story, 24-hour expiration
- Daily: 100,000,000 × 1 MB = 100,000,000 MB = 100 TB
- Active storage: 100 TB (constant, since stories expire after 24 hours)
- **DOC MATCHES:** Storage calculations verified ✅

### User Data

```
Per user:
- Profile: 2 KB
- Settings: 1 KB
- Following list: 200 users × 8 bytes = 1.6 KB
- Followers list (reference): 8 bytes
- Total: ~5 KB

Total user data:
= 2B × 5 KB
= 10 TB
```

### Feed Cache

```
Per user feed (precomputed):
- 500 post IDs × 8 bytes = 4 KB
- Active users: 500M

Feed cache size:
= 500M × 4 KB
= 2 TB
```

### Metadata Storage

```
Per photo metadata:
- Post ID, user ID, timestamps: 50 bytes
- Caption: 200 bytes
- Location: 50 bytes
- Hashtags: 100 bytes
- Image URLs: 500 bytes
- Counts: 20 bytes
- Total: ~1 KB

Total photo metadata:
= 50M photos/day × 365 × 5 years × 1 KB
= 91 TB
```

### Total Storage

| Component                | Size         |
| ------------------------ | ------------ |
| Photos (5 years)         | 228 PB       |
| Stories (active)         | 100 TB       |
| User data                | 10 TB        |
| Feed cache               | 2 TB         |
| Photo metadata           | 91 TB        |
| **Total**                | **~230 PB**  |

---

## Bandwidth Estimation

### Upload Bandwidth

```
Peak uploads: 3,000/second
Average size: 2.5 MB (with all resolutions)

Upload bandwidth:
= 3,000 × 2.5 MB
= 7.5 GB/second
= 60 Gbps
```

### Feed Download Bandwidth

```
Feed request: 20 photos × 100 KB (medium size)
= 2 MB per feed load

Peak feed requests: 180K QPS

Feed bandwidth:
= 180K × 2 MB
= 360 GB/second
= 2.88 Tbps
```

### Story Bandwidth

```
Story view: 1 MB average
Peak story views: 60K/second

Story bandwidth:
= 60K × 1 MB
= 60 GB/second
= 480 Gbps
```

### Total Bandwidth

```
Upload: 60 Gbps
Feed: 2.88 Tbps
Stories: 480 Gbps
Other: 200 Gbps
────────────────
Total: ~3.6 Tbps peak
```

---

## Infrastructure Sizing

### Application Servers

```
API servers:
- Handle 5,000 QPS per server
- Total QPS: 300K (feed + stories + uploads + other)
- Servers needed: 300K / 5K = 60 servers
- With redundancy (3x): 180 servers
```

### Image Processing Workers

```
Peak uploads: 3,000/second
Processing time: 5 seconds per photo
Workers needed: 3,000 × 5 = 15,000 concurrent jobs

With 10 jobs per worker:
= 1,500 workers

With headroom (2x):
= 3,000 image processing workers
```

### Feed Service

```
Peak feed QPS: 180K
Per server: 10K QPS

Feed servers:
= 180K / 10K
= 18 servers

With redundancy (3x):
= 54 feed servers
```

### Storage Servers

```
Total storage: 230 PB
Per server: 100 TB

Storage servers:
= 230 PB / 100 TB
= 2,300 servers

With replication (3x):
= 6,900 storage servers
```

### Cache (Redis)

```
Feed cache: 2 TB
User sessions: 500 GB
Hot data: 1 TB
Total: 3.5 TB

Redis cluster:
- 64 GB per node
- Nodes needed: 3.5 TB / 50 GB usable = 70 nodes
- With replication: 210 nodes
```

### CDN

```
Bandwidth: 3.6 Tbps
Per edge server: 100 Gbps

Edge servers:
= 3.6 Tbps / 100 Gbps
= 36 servers

Distributed across 100+ PoPs globally
```

---

## Cost Estimation (Monthly)

### Storage

| Component              | Size       | Unit Cost  | Monthly Cost |
| ---------------------- | ---------- | ---------- | ------------ |
| Hot storage (30 days)  | 3.75 PB    | $0.023/GB  | $86,250      |
| Warm storage (1 year)  | 45.6 PB    | $0.0125/GB | $570,000     |
| Cold storage (archive) | 180 PB     | $0.004/GB  | $720,000     |
| Redis cache            | 3.5 TB     | $0.15/GB   | $525         |
| **Storage Total**      |            |            | **$1,376,775**|

### Compute

| Component              | Units      | Unit Cost  | Monthly Cost |
| ---------------------- | ---------- | ---------- | ------------ |
| API servers            | 180        | $400       | $72,000      |
| Feed servers           | 54         | $500       | $27,000      |
| Image workers          | 3,000      | $200       | $600,000     |
| Database servers       | 100        | $1,000     | $100,000     |
| **Compute Total**      |            |            | **$799,000** |

### CDN/Bandwidth

| Component              | Volume     | Unit Cost  | Monthly Cost |
| ---------------------- | ---------- | ---------- | ------------ |
| CDN bandwidth          | 9.7 EB     | $0.02/GB   | $194,000,000 |
| Origin egress          | 500 PB     | $0.05/GB   | $25,000,000  |
| **Bandwidth Total**    |            |            | **$219,000,000**|

### Total Monthly Cost

```
Storage:    $1,376,775
Compute:    $799,000
Bandwidth:  $219,000,000
Other:      $2,000,000 (monitoring, misc)
─────────────────────────
Total:      ~$223,000,000/month

Cost per DAU:
= $223M / 500M
= $0.45 per DAU per month

Cost per photo upload:
= $223M / (50M × 30)
= $0.15 per photo
```

---

## Scaling Considerations

### Horizontal Scaling Triggers

| Metric                  | Threshold  | Action                        |
| ----------------------- | ---------- | ----------------------------- |
| API latency P99         | > 500ms    | Add API servers               |
| Feed latency P99        | > 300ms    | Add feed servers              |
| Image queue depth       | > 10,000   | Add image workers             |
| Cache hit rate          | < 90%      | Increase Redis cluster        |
| Storage utilization     | > 80%      | Add storage nodes             |

### Traffic Patterns

| Event                   | Traffic Multiplier | Duration    |
| ----------------------- | ------------------ | ----------- |
| Normal                  | 1x                 | -           |
| New Year's Eve          | 5x                 | 4 hours     |
| Major event (Olympics)  | 3x                 | 2 weeks     |
| Celebrity post          | 2x (localized)     | 1-2 hours   |

### Capacity Planning

| Timeframe | DAU    | Daily Uploads | Storage/Year |
| --------- | ------ | ------------- | ------------ |
| Current   | 500M   | 50M           | 45.6 PB      |
| +1 year   | 650M   | 65M           | 59.3 PB      |
| +2 years  | 800M   | 80M           | 73 PB        |

---

## Summary

| Metric                | Value                |
| --------------------- | -------------------- |
| Peak upload QPS       | 3,000                |
| Peak feed QPS         | 180K                 |
| Daily photo storage   | 125 TB               |
| Total storage (5yr)   | 230 PB               |
| Peak bandwidth        | 3.6 Tbps             |
| API servers           | 180                  |
| Image workers         | 3,000                |
| Monthly cost          | ~$223M               |
| Cost per DAU          | $0.45                |

