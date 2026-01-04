# Recommendation System - Capacity Estimation

## Traffic Estimation

### User Base

| Metric              | Value         | Notes                              |
| ------------------- | ------------- | ---------------------------------- |
| Total users         | 500M          | Registered accounts                |
| Daily active users  | 100M          | 20% DAU ratio                      |
| Sessions per user   | 3/day         | Average                            |
| Recommendations/session | 50        | Multiple recommendation widgets    |

### Request Volume

```
Daily recommendation requests:
= DAU × Sessions × Recommendations per session
= 100M × 3 × 50
= 15 billion requests/day

Average QPS:
= 15B / 86,400
= 173,611 QPS
≈ 175K QPS

Peak QPS (3x average):
= 175K × 3
= 525K QPS
≈ 500K QPS
```

**Math Verification:**
- Assumptions: 15B requests/day, 86,400 seconds/day, 3x peak multiplier
- Average QPS: 15,000,000,000 / 86,400 = 173,611.11 QPS ≈ 175,000 QPS (rounded)
- Peak QPS: 175,000 × 3 = 525,000 QPS ≈ 500,000 QPS (rounded)
- **DOC MATCHES:** QPS calculations verified ✅

### Request Types

| Request Type           | Percentage | QPS (Peak) |
| ---------------------- | ---------- | ---------- |
| Homepage recommendations | 40%       | 200K       |
| Similar items           | 30%       | 150K       |
| Personalized search     | 20%       | 100K       |
| "Because you..." rows   | 10%       | 50K        |

---

## Storage Estimation

### User Data

```
Per user:
- User ID: 8 bytes
- User embedding (128 dimensions): 128 × 4 bytes = 512 bytes
- Interaction history (last 1000 items): 1000 × 8 bytes = 8 KB
- Preferences/features: 1 KB
- Total per user: ~10 KB

Total user data:
= 500M users × 10 KB
= 5 TB
```

**Math Verification:**
- Assumptions: 500M users, 10 KB per user
- Calculation: 500,000,000 × 10 KB = 5,000,000,000 KB = 5 TB
- **DOC MATCHES:** Storage calculations verified ✅

### Item Data

```
Per item:
- Item ID: 8 bytes
- Item embedding (128 dimensions): 512 bytes
- Metadata (title, category, tags): 2 KB
- Content features: 1 KB
- Total per item: ~4 KB

Total item data:
= 100M items × 4 KB
= 400 GB
```

**Math Verification:**
- Assumptions: 100M items, 4 KB per item
- Calculation: 100,000,000 × 4 KB = 400,000,000 KB = 400 GB
- **DOC MATCHES:** Storage calculations verified ✅

### Interaction Data

```
Daily interactions:
= 100M DAU × 20 interactions/user
= 2 billion interactions/day

Per interaction:
- User ID: 8 bytes
- Item ID: 8 bytes
- Action type: 1 byte
- Timestamp: 8 bytes
- Context: 20 bytes
- Total: ~50 bytes

Daily interaction storage:
= 2B × 50 bytes
= 100 GB/day

Yearly (with 90-day retention for real-time):
= 100 GB × 90 days
= 9 TB
```

**Math Verification:**
- Assumptions: 2B interactions/day, 50 bytes per interaction, 90-day retention
- Daily: 2,000,000,000 × 50 bytes = 100,000,000,000 bytes = 100 GB
- 90-day: 100 GB × 90 = 9,000 GB = 9 TB
- **DOC MATCHES:** Storage calculations verified ✅

### Model Storage

```
Collaborative filtering model:
- User embeddings: 500M × 512 bytes = 256 GB
- Item embeddings: 100M × 512 bytes = 51 GB
- Total: ~310 GB

Ranking model:
- Neural network weights: ~1 GB
- Feature transformations: ~500 MB
- Total: ~1.5 GB

Pre-computed recommendations:
- Top 1000 items per active user
- 100M active users × 1000 × 8 bytes = 800 GB
```

### Total Storage

| Component                | Size     |
| ------------------------ | -------- |
| User data                | 5 TB     |
| Item data                | 400 GB   |
| Interaction data (90 days)| 9 TB    |
| User embeddings          | 256 GB   |
| Item embeddings          | 51 GB    |
| Pre-computed recs        | 800 GB   |
| **Total**                | **~16 TB**|

---

## Compute Estimation

### Candidate Generation

```
Per request:
- Approximate nearest neighbor search
- 128-dimension embedding
- Search 100M items
- Return top 1000 candidates

Using FAISS with IVF index:
- Latency: ~5ms
- CPU: 0.1 core-seconds

Peak compute:
= 500K QPS × 0.1 core-seconds
= 50,000 cores
```

### Ranking Model Inference

```
Per request:
- Score 1000 candidates
- Neural network with 3 layers
- ~100 features per candidate

Inference time: ~10ms
GPU utilization: 1 inference = 0.01 GPU-seconds

Peak GPU compute:
= 500K QPS × 0.01 GPU-seconds
= 5,000 GPUs

With batching (10 requests/batch):
= 500 GPUs
```

### Batch Training

```
Daily training data:
= 2B interactions

Model training:
- Matrix factorization: 4 hours on 100 GPUs
- Deep learning ranking: 8 hours on 50 GPUs
- Total: ~12 GPU-hours × 100 = 1,200 GPU-hours/day
```

---

## Bandwidth Estimation

### Inbound (User Signals)

```
Per interaction event:
- User ID, item ID, action, context
- ~200 bytes

Daily inbound:
= 2B events × 200 bytes
= 400 GB/day
= 37 Mbps average
= 111 Mbps peak
```

### Outbound (Recommendations)

```
Per recommendation response:
- 50 items × 200 bytes (ID, score, metadata)
= 10 KB

Daily outbound:
= 15B requests × 10 KB
= 150 TB/day

Average bandwidth:
= 150 TB / 86,400 seconds
= 14 Gbps

Peak bandwidth (3x):
= 42 Gbps
```

---

## Infrastructure Sizing

### Recommendation Servers

```
Per server:
- Handle 5,000 QPS
- 32 CPU cores
- 64 GB RAM (for embeddings cache)
- 500 GB SSD

Servers needed:
= 500K QPS / 5K QPS per server
= 100 servers

With redundancy (2x):
= 200 servers
```

### Embedding Index Servers (FAISS)

```
Index size: 100M items × 512 bytes = 51 GB
RAM per server: 128 GB (index + overhead)

Servers needed:
= 51 GB / 100 GB usable
= 1 server (replicated 3x for availability)
= 3 servers per region

With 5 regions:
= 15 embedding servers
```

### GPU Servers (Ranking)

```
Per GPU server:
- 8 GPUs (A100)
- Handle 80,000 QPS with batching

Servers needed:
= 500K QPS / 80K QPS per server
= 7 servers

With redundancy:
= 15 GPU servers
```

### Feature Store (Redis)

```
User features: 500M × 1 KB = 500 GB
Item features: 100M × 2 KB = 200 GB
Total: 700 GB

Redis cluster:
- 64 GB per node
- Nodes needed: 700 GB / 50 GB usable = 14 nodes
- With replication (3x): 42 nodes
```

### Kafka (Event Streaming)

```
Daily events: 2B
Peak events/second: 50,000

Kafka cluster:
- 10 brokers
- 3x replication
- 7-day retention
```

---

## Cost Estimation (Monthly)

### Compute

| Component              | Units      | Unit Cost  | Monthly Cost |
| ---------------------- | ---------- | ---------- | ------------ |
| Recommendation servers | 200        | $500       | $100,000     |
| Embedding servers      | 15         | $1,000     | $15,000      |
| GPU servers            | 15         | $15,000    | $225,000     |
| Training GPUs          | 1,200 hrs  | $3/hr      | $3,600       |
| **Compute Total**      |            |            | **$343,600** |

### Storage

| Component              | Size       | Unit Cost  | Monthly Cost |
| ---------------------- | ---------- | ---------- | ------------ |
| User/Item data (SSD)   | 6 TB       | $0.10/GB   | $600         |
| Interaction data       | 9 TB       | $0.02/GB   | $180         |
| Model storage          | 1 TB       | $0.10/GB   | $100         |
| Redis cluster          | 700 GB     | $0.15/GB   | $105         |
| **Storage Total**      |            |            | **$985**     |

### Network

| Component              | Volume     | Unit Cost  | Monthly Cost |
| ---------------------- | ---------- | ---------- | ------------ |
| Outbound bandwidth     | 4.5 PB     | $0.05/GB   | $225,000     |
| Inter-region           | 500 TB     | $0.02/GB   | $10,000      |
| **Network Total**      |            |            | **$235,000** |

### Total Monthly Cost

```
Compute:  $343,600
Storage:  $985
Network:  $235,000
Other:    $20,000 (monitoring, logging)
─────────────────────
Total:    ~$600,000/month

Cost per 1M recommendations:
= $600,000 / (15B × 30 days / 1M)
= $600,000 / 450,000
= $1.33 per 1M recommendations
```

---

## Scaling Considerations

### Horizontal Scaling Triggers

| Metric                  | Threshold  | Action                        |
| ----------------------- | ---------- | ----------------------------- |
| CPU utilization         | > 70%      | Add recommendation servers    |
| Latency P99             | > 80ms     | Add servers or optimize       |
| GPU utilization         | > 80%      | Add GPU servers               |
| Cache hit rate          | < 90%      | Increase Redis cluster        |
| Kafka consumer lag      | > 1 minute | Add consumers                 |

### Capacity Planning

| Timeframe | Users  | Items  | QPS   | Servers |
| --------- | ------ | ------ | ----- | ------- |
| Current   | 500M   | 100M   | 500K  | 230     |
| +1 year   | 750M   | 150M   | 750K  | 345     |
| +2 years  | 1B     | 200M   | 1M    | 460     |

---

## Summary

| Metric                | Value                |
| --------------------- | -------------------- |
| Peak QPS              | 500K                 |
| Total storage         | 16 TB                |
| Recommendation servers| 200                  |
| GPU servers           | 15                   |
| Redis nodes           | 42                   |
| Monthly cost          | ~$600K               |
| Cost per 1M recs      | $1.33                |

