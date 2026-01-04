# Centralized Logging System - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, failure scenarios, and level-specific expectations.

---

## Trade-off Questions

### Q1: Why use Elasticsearch instead of regular database?

**Answer:**

**Regular Database (PostgreSQL):**
- Not optimized for full-text search
- Slow queries for text search
- Poor compression
- High storage costs

**Elasticsearch (Chosen):**
- Optimized for full-text search
- Efficient compression
- Fast text search queries
- Better for log data

**Key insight:** Elasticsearch is optimized for the search patterns of log data.

---

### Q2: How do you handle log volume at scale?

**Answer:**

**Strategies:**

1. **Compression**: 4:1 compression ratio reduces storage
2. **Retention Policies**: Hot storage (7 days), cold storage (90 days)
3. **Tiered Storage**: Move old logs to cheaper storage (S3 Glacier)
4. **Indexing**: Index only important fields (reduce index size)
5. **Sampling**: Sample verbose logs (optional)

---

### Q3: How would you scale from 100M to 1B logs/second?

**Answer:**

1. **Linear Scaling**
   ```
   Collectors: 200 → 2,000
   Search services: 100 → 1,000
   ```

2. **Storage Scaling**
   ```
   Elasticsearch: Scale to 10x nodes
   Storage: 122 PB → 1,220 PB
   ```

3. **Architecture Changes**
   - Regional deployment (collectors close to services)
   - Better compression (reduce storage 2x)
   - Tiered storage (hot/cold/warm)
   - Log sampling (reduce volume)

---

## Failure Scenarios

### Scenario 1: Elasticsearch Cluster Failure

**Impact:** Logs cannot be stored, searches fail

**Recovery:**
1. Logs buffered in Kafka (1-day retention)
2. Restore from backup or rebuild cluster
3. Replay logs from Kafka
4. System recovers

---

## Level-Specific Expectations

### L4 (Entry Level)

**Q: How does centralized logging work?**
- Services send logs to collectors
- Logs stored in searchable storage
- Users search logs via UI
- Real-time streaming available

### L5 (Mid Level)

**Q: How do you optimize storage costs?**
- Compression (reduce size 4x)
- Retention policies (hot/cold storage)
- Tiered storage (cheaper for old data)
- Index optimization

### L6 (Senior Level)

**Q: How do you design for global scale?**
- Regional deployment
- Multi-region replication
- Cost optimization strategies
- Performance optimization

---

## Summary

Key takeaways:
- Elasticsearch optimized for log search
- Compression and retention policies reduce costs
- Kafka buffering handles traffic spikes
- Storage costs dominate (optimize aggressively)

