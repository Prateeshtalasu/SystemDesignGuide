# Payment System - Capacity Estimation

## Overview

This document calculates the infrastructure requirements for a payment system processing 10 million transactions per day with a transaction volume of $1 billion per month.

---

## Traffic Estimation

### Transaction Rate

**Given:**
- Transactions per day: 10 million
- Operating 24/7
- Transaction volume: $1 billion/month

**Calculations:**

```
Transactions per day = 10 million
Transactions per hour = 10M / 24 hours
                     = 416,667 transactions/hour

Transactions per second (average) = 10M / 86,400 seconds
                                 = 115.7 transactions/second
                                 ≈ 115 QPS

Peak traffic (10x average during Black Friday, holiday sales):
Peak QPS = 115 × 10 = 1,150 QPS
```

**Math Verification:**
- Assumptions: 10M transactions/day, 86,400 seconds/day, 10x peak multiplier
- Average QPS: 10,000,000 / 86,400 = 115.74 transactions/sec ≈ 115 QPS (rounded)
- Peak QPS: 115 × 10 = 1,150 QPS
- **DOC MATCHES:** Transaction QPS calculations verified ✅
```

### Request Type Distribution

**Breakdown by operation:**

| Operation Type | % of Requests | QPS (Average) | QPS (Peak) |
|----------------|---------------|---------------|------------|
| Payment Processing | 60% | 69 | 690 |
| Refunds | 10% | 12 | 115 |
| Reconciliation | 5% | 6 | 58 |
| Fraud Checks | 20% | 23 | 230 |
| Payment Method Management | 3% | 3 | 35 |
| Webhooks | 2% | 2 | 23 |
| **Total** | **100%** | **115** | **1,151** |

### Transaction Value Distribution

**Average transaction value:**
```
Monthly volume: $1 billion
Transactions per month: 10M × 30 = 300M

Average transaction value = $1B / 300M
                         = $3.33 per transaction
```

**Math Verification:**
- Assumptions: $1B monthly volume, 300M transactions/month (10M/day × 30 days)
- Calculation: $1,000,000,000 / 300,000,000 = $3.333 per transaction ≈ $3.33
- **DOC MATCHES:** Average transaction value verified ✅
```

**Transaction value breakdown:**
- Micro-transactions (< $10): 40% of transactions, 5% of volume
- Small ($10-$100): 45% of transactions, 30% of volume
- Medium ($100-$1,000): 12% of transactions, 40% of volume
- Large (> $1,000): 3% of transactions, 25% of volume

---

## Storage Estimation

### Transaction Storage

**Transaction Data:**
```
Transactions per day: 10 million
Data per transaction:
  - Transaction ID: 50 bytes
  - Payment method ID: 50 bytes
  - Amount: 8 bytes
  - Currency: 3 bytes
  - Status: 20 bytes
  - Timestamps: 16 bytes
  - Metadata (JSON): 1,000 bytes
  - Payment processor response: 500 bytes
  - Fraud check result: 200 bytes
  - Audit trail: 200 bytes
  Total: ~2 KB per transaction

Daily storage = 10M × 2 KB
              = 20 GB/day

**Math Verification:**
- Assumptions: 10M transactions/day, 2 KB per transaction
- Calculation: 10,000,000 × 2 KB = 20,000,000 KB = 20 GB
- **DOC MATCHES:** Storage calculations verified ✅
```

**Retention Requirements:**
```
Regulatory retention: 7 years (PCI DSS, SOX compliance)
Total storage = 20 GB × 365 days × 7 years
              = 51.1 TB

With 3x replication (backup, disaster recovery):
Total storage = 51.1 TB × 3 = 153.3 TB

**Math Verification:**
- Assumptions: 20 GB/day, 365 days/year, 7 years retention, 3x replication
- Yearly: 20 GB × 365 = 7,300 GB = 7.3 TB
- 7-year: 7.3 TB × 7 = 51.1 TB
- With replication: 51.1 TB × 3 = 153.3 TB
- **DOC MATCHES:** Storage calculations verified ✅
```

### Ledger Storage (Double-Entry Bookkeeping)

**Ledger Entries:**
```
Entries per transaction: 2 (debit + credit)
Entries per day: 10M × 2 = 20 million entries

Data per ledger entry:
  - Entry ID: 8 bytes
  - Transaction ID: 50 bytes
  - Account ID: 50 bytes
  - Debit amount: 8 bytes
  - Credit amount: 8 bytes
  - Balance: 8 bytes
  - Timestamp: 8 bytes
  - Description: 200 bytes
  - Metadata: 200 bytes
  Total: ~540 bytes per entry

Daily storage = 20M × 540 bytes
              = 10.8 GB/day

7-year retention = 10.8 GB × 365 × 7
                 = 27.6 TB

With replication = 27.6 TB × 3 = 82.8 TB
```

### Payment Method Storage

**Stored Payment Methods:**
```
Users: 50 million (estimated)
Payment methods per user: 2 (average)
Total payment methods: 100 million

Data per payment method:
  - Payment method ID: 50 bytes
  - User ID: 50 bytes
  - Type (card/ACH/wallet): 20 bytes
  - Last 4 digits: 4 bytes
  - Expiry date: 8 bytes
  - Billing address: 200 bytes
  - Token (encrypted): 100 bytes
  - Metadata: 200 bytes
  Total: ~632 bytes per payment method

Total storage = 100M × 632 bytes
              = 63.2 GB
```

### Fraud Check Storage

**Fraud Check Results:**
```
Fraud checks per day: 10M (one per transaction)
Data per fraud check:
  - Check ID: 50 bytes
  - Transaction ID: 50 bytes
  - Risk score: 4 bytes
  - Rules triggered: 200 bytes
  - ML model predictions: 500 bytes
  - Timestamp: 8 bytes
  Total: ~812 bytes per check

Daily storage = 10M × 812 bytes
              = 8.1 GB/day

Retention: 2 years (fraud investigation)
Total storage = 8.1 GB × 365 × 2
              = 5.9 TB
```

### Idempotency Key Storage

**Idempotency Keys:**
```
Keys per day: 10M (one per transaction attempt)
Data per key:
  - Key: 100 bytes (UUID)
  - Transaction ID: 50 bytes
  - Response (cached): 2 KB
  - Timestamp: 8 bytes
  Total: ~2.2 KB per key

Daily storage = 10M × 2.2 KB
              = 22 GB/day

Retention: 24 hours (TTL)
Active storage = 22 GB (rolling window)
```

### Total Storage Summary

| Component           | Size       | Notes                           |
| ------------------- | ---------- | ------------------------------- |
| Transactions (7y)   | 153.3 TB   | With 3x replication             |
| Ledger entries (7y) | 82.8 TB    | With 3x replication             |
| Payment methods     | 63.2 GB    | Active data                     |
| Fraud checks (2y)   | 5.9 TB     | Investigation retention         |
| Idempotency keys    | 22 GB      | 24-hour rolling window          |
| **Total**           | **~243 TB**| Primary storage                 |

---

## Bandwidth Estimation

### API Request Bandwidth

**Incoming Requests:**
```
Average request size: 2 KB (payment data, metadata)
Peak QPS: 1,150

Incoming bandwidth = 1,150 × 2 KB/s
                   = 2.3 MB/s
                   = 18.4 Mbps

Peak incoming: 18.4 Mbps
```

### API Response Bandwidth

**Outgoing Responses:**
```
Average response size: 1 KB (transaction confirmation)
Peak QPS: 1,150

Outgoing bandwidth = 1,150 × 1 KB/s
                   = 1.15 MB/s
                   = 9.2 Mbps

Peak outgoing: 9.2 Mbps
```

### Payment Processor Communication

**To Payment Processors:**
```
Requests to processors: 69 QPS (average)
Request size: 1 KB
Response size: 2 KB

Outgoing to processors = 69 × 1 KB/s = 69 KB/s = 0.55 Mbps
Incoming from processors = 69 × 2 KB/s = 138 KB/s = 1.1 Mbps
```

### Total Bandwidth

| Direction       | Bandwidth | Notes                    |
| --------------- | --------- | ------------------------ |
| API incoming    | 18.4 Mbps | Client requests          |
| API outgoing    | 9.2 Mbps  | Client responses         |
| Processor comm  | 1.65 Mbps | Payment processor API    |
| **Total**       | **~30 Mbps** | Peak requirement         |

---

## Memory Estimation

### Cache Requirements

**Transaction Cache (Redis):**
```
Hot transactions (last 24 hours): 10M
Data per transaction: 2 KB
Cache size = 10M × 2 KB = 20 GB
```

**Idempotency Cache (Redis):**
```
Active idempotency keys: 10M (24-hour TTL)
Data per key: 2.2 KB
Cache size = 10M × 2.2 KB = 22 GB
```

**Payment Method Cache (Redis):**
```
Active payment methods: 100M
Data per method: 632 bytes
Cache size = 100M × 632 bytes = 63.2 GB
```

**Fraud Check Cache (Redis):**
```
Recent fraud checks (1 hour): 416,667
Data per check: 812 bytes
Cache size = 416,667 × 812 bytes = 338 MB
```

### Total Memory Requirements

| Component              | Memory    | Notes                      |
| ---------------------- | --------- | -------------------------- |
| Transaction cache      | 20 GB     | Per Redis cluster          |
| Idempotency cache      | 22 GB     | Per Redis cluster          |
| Payment method cache   | 63.2 GB   | Per Redis cluster          |
| Fraud check cache      | 338 MB    | Per Redis cluster          |
| **Total Redis**        | **~105 GB**| Distributed across cluster |
| Application memory     | 32 GB     | Per application server     |

---

## Server Estimation

### Payment Processing Service

**Calculation:**
```
Payment QPS: 69 (average), 690 (peak)
Transactions per server: 100/second (limited by payment processor API)

Servers needed (average) = 69 / 100 = 1 server
Servers needed (peak) = 690 / 100 = 7 servers
With redundancy: 10 servers
```

**Payment Server Specs:**
```
CPU: 16 cores (payment processing, encryption)
Memory: 32 GB
Network: 10 Gbps
Disk: 500 GB SSD
```

### Fraud Detection Service

**Calculation:**
```
Fraud check QPS: 23 (average), 230 (peak)
Checks per server: 50/second (ML model inference)

Servers needed = 230 / 50 = 5 servers
With redundancy: 8 servers
```

**Fraud Server Specs:**
```
CPU: 32 cores (ML inference is CPU-intensive)
Memory: 64 GB (model loading)
Network: 10 Gbps
Disk: 1 TB SSD (model storage)
```

### Ledger Service

**Calculation:**
```
Ledger writes: 20M entries/day = 231 entries/second
Writes per server: 1,000/second

Servers needed = 231 / 1,000 = 1 server
With redundancy: 3 servers
```

**Ledger Server Specs:**
```
CPU: 16 cores
Memory: 64 GB (for transaction processing)
Network: 10 Gbps
Disk: 2 TB SSD
```

### Server Summary

| Component           | Servers | Specs                           |
| ------------------- | ------- | ------------------------------- |
| Payment service     | 10      | 16 cores, 32 GB RAM, 500 GB SSD |
| Fraud service       | 8       | 32 cores, 64 GB RAM, 1 TB SSD  |
| Ledger service      | 3       | 16 cores, 64 GB RAM, 2 TB SSD  |
| Redis cluster       | 6       | 8 cores, 128 GB RAM, 1 TB SSD  |
| PostgreSQL (RDS)    | 3       | 16 cores, 256 GB RAM, 20 TB    |
| **Total**           | **30**  | Plus managed services           |

---

## Cost Estimation

### Compute Costs (AWS)

| Component           | Instance Type | Count | Monthly Cost |
| ------------------- | ------------- | ----- | ------------ |
| Payment service     | c6i.4xlarge   | 10    | $3,000       |
| Fraud service       | c6i.8xlarge   | 8     | $4,800       |
| Ledger service      | r6i.4xlarge   | 3     | $1,350       |
| Redis cluster       | r6i.2xlarge   | 6     | $2,160       |
| PostgreSQL (RDS)    | db.r6i.4xlarge| 3     | $4,500       |
| **Compute Total**   |               |       | **$15,810**  |

### Storage Costs

| Type                | Size/Requests | Monthly Cost |
| ------------------- | ------------- | ------------ |
| EBS (SSD)           | 243 TB        | $24,300      |
| S3 (backup)         | 243 TB        | $5,460       |
| **Storage Total**   |               | **$29,760**  |

### Payment Processor Fees

```
Transactions per month: 10M × 30 = 300M
Average transaction: $3.33

Processor fees:
- Percentage: 2.9% of transaction value
- Fixed: $0.30 per transaction

Monthly fees:
Percentage = $1B × 0.029 = $29M
Fixed = 300M × $0.30 = $90M
Total processor fees = $119M/month
```

**Note:** Processor fees dominate costs. Enterprise pricing can reduce by 20-30%.

### Network Costs

```
Data transfer: 30 Mbps peak = minimal
Cross-AZ: ~$500/month

Network total: ~$500/month
```

### Total Monthly Cost

| Category    | Cost         |
| ----------- | ------------ |
| Compute     | $15,810     |
| Storage     | $29,760     |
| Network     | $500        |
| Processor fees | $119,000,000 |
| Misc (20%)  | $23,800,000 |
| **Total**   | **~$143.6M/month** |

**With enterprise processor pricing (20% discount):**
- Processor fees: $95.2M/month
- Total: ~$119.8M/month

### Cost per Transaction

```
Monthly transactions: 300 million
Monthly cost: $119.8M (with enterprise pricing)

Cost per transaction = $119.8M / 300M
                     = $0.40 per transaction

Infrastructure cost: $0.00005 per transaction
Processor cost: $0.317 per transaction
```

---

## Scaling Projections

### 10x Scale (100M transactions/day)

| Metric          | Current     | 10x Scale   |
| --------------- | ----------- | ----------- |
| Transactions/day| 10M        | 100M        |
| QPS (average)   | 115         | 1,150       |
| QPS (peak)      | 1,150       | 11,500      |
| Servers         | 30          | 300         |
| Storage         | 243 TB      | 2.43 PB     |
| Monthly cost    | $119.8M     | $1.2B       |

### Bottlenecks at Scale

1. **Payment processor API limits**: Rate limits from processors
   - Solution: Multiple processor accounts, load balancing

2. **Database write throughput**: Ledger writes are sequential
   - Solution: Sharding by transaction ID, parallel writes

3. **Fraud check latency**: ML inference is CPU-intensive
   - Solution: GPU acceleration, model optimization

4. **Idempotency key storage**: Memory requirements grow
   - Solution: Shard Redis cluster, reduce TTL

---

## Quick Reference Numbers

### Interview Mental Math

```
10 million transactions/day
115 QPS average
1,150 QPS peak
243 TB storage
$119.8M/month (with enterprise pricing)
$0.40 per transaction
```

### Key Ratios

```
Transactions per user: 0.2/day (50M users)
Average transaction: $3.33
Payment success rate: > 99%
Fraud rate: < 0.1%
Cache hit rate: 80% (idempotency)
```

### Time Estimates

```
Payment processing: < 2s p95
Fraud check: < 500ms p95
Ledger write: < 100ms p95
Idempotency check: < 10ms p95
```

---

## Summary

| Aspect              | Value                                  |
| ------------------- | -------------------------------------- |
| Transactions/day    | 10 million                             |
| QPS (average)      | 115                                    |
| QPS (peak)          | 1,150                                  |
| Storage             | 243 TB                                 |
| Servers             | ~30                                    |
| Monthly cost        | ~$119.8M (with enterprise pricing)     |
| Cost per transaction| $0.40                                  |


