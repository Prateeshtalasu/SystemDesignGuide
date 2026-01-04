# Ticket Booking System - Capacity Estimation

## Overview

This document calculates the infrastructure requirements for a ticket booking system handling 1 million bookings per day.

---

## Traffic Estimation

### Booking Volume

**Given:**
- Target: 1 million bookings/day
- Operating 24/7
- Peak traffic: 50x average (popular movie releases, concerts)

**Calculations:**

```
Bookings per day = 1 million
Bookings per hour = 1M / 24 hours = 41,667 bookings/hour
Bookings per minute = 41,667 / 60 = 694 bookings/minute
Bookings per second (average) = 694 / 60 = 12 bookings/second

Peak traffic (50x):
Peak bookings/second = 12 × 50 = 600 bookings/second
With 2x buffer: Target = 1,000 bookings/second
```

### Seat Selection Operations

**Seat Selection Rate:**
```
Seat selection to booking ratio: 5:1 (users explore before booking)
Seat selections per day = 1M × 5 = 5 million selections/day

Seat selections per second (average) = 5M / 86,400 = 58 selections/second
Peak selections/second = 58 × 50 = 2,900 selections/second
```

### Read vs Write Ratio

**Operations Breakdown:**
```
Seat map views: 70% of traffic
Seat selections: 20% of traffic
Bookings: 10% of traffic

Read:Write ratio = 70:30 = 2.3:1
```

### API Request Rates

**Per Operation QPS:**

| Operation | Average QPS | Peak QPS | Calculation |
|-----------|------------|----------|-------------|
| View Seat Map | 700 | 35,000 | 70% of 1,000 ops/sec |
| Select Seats | 200 | 10,000 | 20% of 1,000 ops/sec |
| Lock Seats | 200 | 10,000 | Same as selections |
| Create Booking | 100 | 5,000 | 10% of 1,000 ops/sec |
| Payment Processing | 100 | 5,000 | Same as bookings |

---

## Storage Estimation

### Booking Storage

**Booking Data Size:**
```
Per booking:
  - Booking ID: 16 bytes
  - User ID: 8 bytes
  - Event ID: 8 bytes
  - Show ID: 8 bytes
  - Seat IDs (JSON): 500 bytes (avg 4 seats)
  - Payment ID: 16 bytes
  - Total amount: 8 bytes
  - Status: 1 byte
  - Timestamps: 32 bytes
  Total: ~600 bytes per booking
```

**Booking Storage Requirements:**
```
Bookings per day: 1 million
Bookings per month: 1M × 30 = 30 million bookings
Bookings per year: 1M × 365 = 365 million bookings

Monthly storage = 30M × 600 bytes = 18 GB/month
Annual storage = 365M × 600 bytes = 219 GB/year

With 3x replication = 657 GB/year
```

**Math Verification:**
- Assumptions: 1M bookings/day, 600 bytes per booking, 30 days/month, 365 days/year, 3x replication
- Monthly: 30,000,000 × 600 bytes = 18,000,000,000 bytes = 18 GB
- Annual: 365,000,000 × 600 bytes = 219,000,000,000 bytes = 219 GB
- With replication: 219 GB × 3 = 657 GB
- **DOC MATCHES:** Storage calculations verified ✅

### Seat Lock Storage

**Lock Data Size:**
```
Per lock:
  - Lock ID: 16 bytes
  - Seat ID: 8 bytes
  - User ID: 8 bytes
  - Show ID: 8 bytes
  - Expires at: 8 bytes
  - Status: 1 byte
  Total: ~50 bytes per lock
```

**Lock Storage:**
```
Active locks: 10,000 (concurrent seat selections)
Lock storage = 10,000 × 50 bytes = 500 KB

Lock history (30 days):
Total locks: 5M × 30 = 150 million
Lock history = 150M × 50 bytes = 7.5 GB
```

### Seat Availability Storage

**Availability Data Size:**
```
Per show:
  - Show ID: 8 bytes
  - Total seats: 4 bytes
  - Available seats: 4 bytes
  - Locked seats: 4 bytes
  - Booked seats: 4 bytes
  Total: ~25 bytes per show
```

**Availability Storage:**
```
Active shows: 10,000
Availability storage = 10,000 × 25 bytes = 250 KB
```

### Total Storage Summary

| Component | Size | Notes |
|-----------|------|-------|
| Bookings (monthly) | 18 GB | Before replication |
| Bookings (annual) | 219 GB | Before replication |
| Seat locks (active) | 500 KB | Current locks |
| Seat locks (history) | 7.5 GB | 30-day retention |
| Seat availability | 250 KB | Current shows |
| **Total active** | **~1 MB** | Current data |
| **Total annual** | **~230 GB** | Before replication |

---

## Bandwidth Estimation

### Incoming Traffic (Writes)

**Booking Operations:**
```
Create booking: 100 ops/sec × 1 KB = 100 KB/s = 0.8 Mbps
Lock seats: 200 ops/sec × 500 bytes = 100 KB/s = 0.8 Mbps

Total incoming: 200 KB/s = 1.6 Mbps
Peak (50x): 80 Mbps
```

### Outgoing Traffic (Reads)

**Seat Map Views:**
```
View seat map: 700 ops/sec × 10 KB = 7 MB/s = 56 Mbps
Get booking: 500 ops/sec × 2 KB = 1 MB/s = 8 Mbps

Total outgoing: 8 MB/s = 64 Mbps
Peak (50x): 3.2 Gbps
```

### Total Bandwidth

| Direction | Bandwidth (Average) | Bandwidth (Peak) | Notes |
|-----------|---------------------|------------------|-------|
| Incoming | 1.6 Mbps | 80 Mbps | Writes |
| Outgoing | 64 Mbps | 3.2 Gbps | Reads |
| **Total** | **~66 Mbps** | **~3.3 Gbps** | Peak requirement |

---

## Memory Estimation

### Cache Requirements

**Seat Map Cache (Redis):**
```
Active shows: 10,000
Seat map size: 50 KB per show
Cache size = 10,000 × 50 KB = 500 MB

With 2x buffer: 1 GB
```

**Seat Lock Cache:**
```
Active locks: 10,000
Lock size: 50 bytes
Cache size = 10,000 × 50 bytes = 500 KB
```

**Booking Cache (Recent):**
```
Recent bookings (last 24 hours): 1 million
Booking size: 600 bytes
Cache size = 1M × 600 bytes = 600 MB

With LRU (keep top 100K): 60 MB
```

### Total Memory Requirements

| Component | Memory | Notes |
|-----------|--------|-------|
| Seat map cache | 1 GB | Active shows |
| Seat lock cache | 500 KB | Active locks |
| Booking cache | 60 MB | Recent bookings |
| **Total cache** | **~1.1 GB** | Redis cluster |
| **Per service** | **~2 GB** | Application memory |

---

## Server Estimation

### Booking Service

**Calculation:**
```
Target: 1,000 bookings/second
Bookings per server: 200/second
Servers needed = 1,000 / 200 = 5 servers
With redundancy: 8 servers
```

**Server Specs:**
```
CPU: 8 cores (booking processing)
Memory: 16 GB
Network: 1 Gbps
```

### Seat Service

**Calculation:**
```
Target: 2,900 seat selections/second
Selections per server: 500/second
Servers needed = 2,900 / 500 = 6 servers
With redundancy: 10 servers
```

**Server Specs:**
```
CPU: 8 cores (high throughput)
Memory: 16 GB (caching)
Network: 1 Gbps
```

### Payment Service

**Calculation:**
```
Target: 1,000 payments/second
Payments per server: 100/second (gateway limited)
Servers needed = 1,000 / 100 = 10 servers
With redundancy: 15 servers
```

**Server Specs:**
```
CPU: 4 cores (I/O bound)
Memory: 8 GB
Network: 1 Gbps
```

### Server Summary

| Component | Servers | Specs |
|-----------|---------|-------|
| Booking Service | 8 | 8 cores, 16 GB RAM |
| Seat Service | 10 | 8 cores, 16 GB RAM |
| Payment Service | 15 | 4 cores, 8 GB RAM |
| Redis Cluster | 6 | 8 cores, 32 GB RAM |
| PostgreSQL | 3 | 16 cores, 64 GB RAM |
| **Total** | **42** | Plus managed services |

---

## Cost Estimation

### Compute Costs (AWS)

| Component | Instance Type | Count | Monthly Cost |
|-----------|--------------|-------|--------------|
| Booking Service | c6i.2xlarge | 8 | $1,920 |
| Seat Service | c6i.2xlarge | 10 | $2,400 |
| Payment Service | c6i.xlarge | 15 | $2,250 |
| Redis Cluster | r6i.2xlarge | 6 | $2,160 |
| PostgreSQL | db.r6i.4xlarge | 3 | $3,600 |
| **Compute Total** | | | **$12,330** |

### Storage Costs

| Type | Size | Monthly Cost |
|------|------|--------------|
| S3 Standard (bookings) | 18 GB/month | $0.40 |
| EBS (databases) | 5 TB | $500 |
| **Storage Total** | | **$500** |

### Network Costs

```
Data transfer out: 3.2 Gbps peak = ~$2,000/month
Cross-AZ: ~$500/month

Network total: ~$2,500/month
```

### Total Monthly Cost

| Category | Cost |
|----------|------|
| Compute | $12,330 |
| Storage | $500 |
| Network | $2,500 |
| Misc (20%) | $3,066 |
| **Total** | **~$18,400/month** |

### Cost per Booking

```
Monthly bookings: 1M × 30 = 30 million
Monthly cost: $18,400

Cost per booking = $18,400 / 30M = $0.00061
= $610 per million bookings
```

---

## Scaling Projections

### 10x Scale (10M bookings/day)

| Metric | Current | 10x Scale |
|--------|---------|-----------|
| Bookings/day | 1M | 10M |
| Bookings/second | 1,000 | 10,000 |
| Servers | 42 | 400 |
| Storage/month | 18 GB | 180 GB |
| Monthly cost | $18.4K | $150K |

---

## Quick Reference Numbers

### Interview Mental Math

```
1 million bookings/day
1,000 bookings/second (peak)
5 million seat selections/day
$18.4K/month
$610 per million bookings
```

### Key Ratios

```
Seat selection to booking: 5:1
Read:Write ratio: 2.3:1
Average seats per booking: 4
Booking completion rate: 40%
```

---

## Summary

| Aspect | Value |
|--------|-------|
| Bookings/day | 1 million |
| Peak bookings/second | 1,000 |
| Seat selections/day | 5 million |
| Storage/month | 18 GB |
| Bandwidth (peak) | 3.3 Gbps |
| Servers | 42 |
| Monthly cost | ~$18,400 |
| Cost per booking | $0.00061 |

