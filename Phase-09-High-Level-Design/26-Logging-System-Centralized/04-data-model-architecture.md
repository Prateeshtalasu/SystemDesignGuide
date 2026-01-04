# Centralized Logging System - Data Model & Architecture

## Component Overview

Before looking at diagrams, let's understand each component and why it exists.

### Components Explained

| Component | Purpose | Why It Exists |
|-----------|---------|---------------|
| **Log Collector** | Receives logs from services | High-throughput ingestion endpoint |
| **Log Storage** | Stores logs with timestamps | Persistence for querying |
| **Search Engine** | Executes log queries | Fast retrieval and search |
| **Real-time Streamer** | Streams logs in real-time | Live log viewing |

---

## Database Choices

| Data Type | Database | Rationale |
|-----------|----------|-----------|
| Log Storage | Elasticsearch | Full-text search, time-series optimized |
| Cold Logs | S3 + Index | Cheap storage for historical logs |
| Log Metadata | Elasticsearch | Fast search by service, level, fields |

---

## Consistency Model

**CAP Theorem Tradeoff:**

We choose **Availability + Partition Tolerance (AP)**:
- **Availability**: Logging system must always operate
- **Partition Tolerance**: System continues operating during network partitions
- **Consistency**: Sacrificed (logs may be slightly stale)

**Why AP over CP?**
- Logging is observational (doesn't need strict consistency)
- Better to serve slightly stale logs than fail
- System must always operate (high availability requirement)

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         CENTRALIZED LOGGING SYSTEM                                  │
└─────────────────────────────────────────────────────────────────────────────────────┘

Services (1,000 services)
    │
    │ Logs (HTTP, Syslog, File-based)
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    LOG COLLECTORS (200 instances)                                   │
│                                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Collector 1 │  │  Collector 2 │  │  Collector 3 │  │  Collector N │          │
│  │              │  │              │  │              │  │              │          │
│  │ - Parse      │  │ - Parse      │  │ - Parse      │  │ - Parse      │          │
│  │ - Compress   │  │ - Compress   │  │ - Compress   │  │ - Compress   │          │
│  │ - Buffer     │  │ - Buffer     │  │ - Buffer     │  │ - Buffer     │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
└─────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┘
          │                  │                  │                  │
          └──────────────────┼──────────────────┼──────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  Kafka (logs)   │
                    │  (Buffering)    │
                    └────────┬────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         LOG STORAGE                                                  │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         HOT STORAGE (7 days)                                │    │
│  │                                                                              │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                      │    │
│  │  │ Elasticsearch│  │ Elasticsearch│  │ Elasticsearch│                      │    │
│  │  │  Cluster     │  │  Cluster     │  │  Cluster     │                      │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘                      │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         COLD STORAGE (90 days)                              │    │
│  │                                                                              │    │
│  │  ┌──────────────┐  ┌──────────────┐                                        │    │
│  │  │  S3 (Logs)   │  │ Elasticsearch│                                        │    │
│  │  │  (Compressed)│  │  (Metadata)  │                                        │    │
│  │  └──────────────┘  └──────────────┘                                        │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    SEARCH ENGINE (100 instances)                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                             │
│  │ Search Eng 1 │  │ Search Eng 2 │  │ Search Eng N │                             │
│  │              │  │              │  │              │                             │
│  │ - Execute    │  │ - Execute    │  │ - Execute    │                             │
│  │ - Search     │  │ - Search     │  │ - Search     │                             │
│  │ - Cache      │  │ - Cache      │  │ - Cache      │                             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                             │
└─────────┼──────────────────┼──────────────────┼──────────────────────────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  Search UI      │
                    │  (Kibana/UI)    │
                    └─────────────────┘
```

---

## Request Flow: Log Ingestion

```
Step 1: Service Emits Log
┌─────────────────────────────────────────────────────────────┐
│ Order Service:                                              │
│ - Logs: ERROR Payment processing failed                    │
│ - Sends to collector via HTTP                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Collector Receives
┌─────────────────────────────────────────────────────────────┐
│ Collector:                                                  │
│ - Parses log format                                         │
│ - Adds metadata (service, timestamp)                        │
│ - Compresses log data                                       │
│ - Buffers logs (batch for 10ms)                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Kafka Buffering
┌─────────────────────────────────────────────────────────────┐
│ Kafka Topic: logs                                           │
│ Partition: hash(service) % 200                              │
│ - Buffers logs before storage                               │
│ - 1-day retention for replay                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Storage
┌─────────────────────────────────────────────────────────────┐
│ Elasticsearch:                                              │
│ - Stores logs with timestamps                               │
│ - Indexes by service, level, fields                        │
│ - Supports full-text search                                │
└─────────────────────────────────────────────────────────────┘
```

---

## Request Flow: Log Search

```
Step 1: User Searches Logs
┌─────────────────────────────────────────────────────────────┐
│ UI: Search "Payment processing failed" service=order-service│
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Search Engine
┌─────────────────────────────────────────────────────────────┐
│ Search Engine:                                              │
│ - Parses search query                                       │
│ - Checks cache (Redis) - MISS                               │
│ - Executes query against Elasticsearch                      │
│ - Returns matching logs                                     │
│ - Caches result (1-minute TTL)                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Sharding Strategy

**Service-Based Sharding:**

- **Kafka Partitions**: `hash(service_name) % partitions` (distributes logs across partitions)
- **Elasticsearch**: Time-based sharding + service hash (distributes across nodes)

**Benefits:**
- Even distribution of logs
- Parallel query execution
- Efficient storage

### Hot Partition Mitigation

**Problem:** If one service generates 10x more logs than others, its partition becomes a bottleneck.

**Example Scenario:**
```
Normal service: 100K logs/second → Partition 45
High-volume service: 1M logs/second → Partition 45 (same partition!)
Result: Partition 45 receives 10x traffic, becomes bottleneck
```

**Detection:**
- Monitor partition lag per partition
- Alert when partition lag > 2x average
- Track partition throughput metrics

**Mitigation Strategies:**

1. **Partition Rebalancing:**
   - If hot partition detected, increase partition count
   - Rebalance existing partitions (Kafka supports this)
   - Distribute hot service logs across multiple partitions

2. **Consumer Scaling:**
   - Add more consumers for hot partitions
   - Auto-scale consumers based on partition lag
   - Each consumer handles subset of hot partition

3. **Rate Limiting:**
   - Rate limit per service (prevent single service from overwhelming)
   - Throttle high-volume services
   - Buffer logs locally if rate limit exceeded

4. **Pre-warming:**
   - Pre-populate cache for popular search queries
   - Pre-index logs from high-volume services
   - Reduce load during peak times

**Implementation:**

```java
@Service
public class HotPartitionDetector {
    
    public void detectAndMitigate() {
        Map<Integer, Long> partitionLag = getPartitionLag();
        double avgLag = partitionLag.values().stream()
            .mapToLong(Long::longValue)
            .average()
            .orElse(0);
        
        partitionLag.entrySet().stream()
            .filter(e -> e.getValue() > avgLag * 2)
            .forEach(e -> {
                int partition = e.getKey();
                long lag = e.getValue();
                
                // Alert ops team
                alertOps("Hot partition detected: " + partition + ", lag: " + lag);
                
                // Auto-scale consumers for this partition
                scaleConsumers(partition, lag);
            });
    }
}
```

**Monitoring:**
- Partition lag per partition (Kafka metrics)
- Partition throughput (logs/second per partition)
- Consumer lag per partition
- Alert threshold: Lag > 2x average for 5 minutes

---

## Replication Strategy

**Storage Replication:**

- **Elasticsearch**: 3 replicas per shard (high availability)
- **S3**: Cross-region replication (disaster recovery)

**Failover:**
- Elasticsearch: Automatic failover to replica
- S3: Read from replica region if primary fails

---

## Summary

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Storage | Elasticsearch (hot), S3 (cold) | Full-text search + cheap archival |
| Sharding | Service hash + time-based | Even distribution + efficient queries |
| Replication | 3x (Elasticsearch), cross-region (S3) | High availability |
| Consistency | Eventual | Observational data, slight staleness acceptable |

