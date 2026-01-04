# Monitoring & Alerting System - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, failure scenarios, and level-specific expectations.

---

## Trade-off Questions

### Q1: Why use time-series database instead of regular database?

**Answer:**

**Regular Database (PostgreSQL):**
- Not optimized for time-series data
- Slow queries for time ranges
- Poor compression
- High storage costs

**Time-Series Database (Chosen):**
- Optimized for time-series queries
- Efficient compression (similar values)
- Fast time-range queries
- Lower storage costs

**Key insight:** Time-series databases are optimized for the access patterns of metrics data.

---

### Q2: Why pull (Prometheus) vs push (StatsD) model?

**Answer:**

**Push Model (StatsD):**
- Services push metrics to collector
- Pros: Simple, works with any service
- Cons: Services must know collector endpoint, harder to scale

**Pull Model (Prometheus):**
- Collector scrapes metrics from services
- Pros: Centralized configuration, service discovery, better scaling
- Cons: Requires metrics endpoint on services

**Hybrid Approach (Chosen):**
- Pull model for standard services (Prometheus)
- Push model for legacy services (StatsD)
- Best of both worlds

---

### Q3: How do you handle alert fatigue?

**Answer:**

**Problems:**
- Too many alerts (100s per day)
- False positives
- Duplicate alerts

**Solutions:**

1. **Alert Grouping**: Group alerts by labels (same issue, different instances)
2. **Alert Deduplication**: Prevent duplicate alerts
3. **Alert Thresholds**: Tune thresholds to reduce false positives
4. **Alert Routing**: Route alerts to appropriate teams
5. **Alert Suppression**: Suppress alerts during maintenance windows

---

### Q4: How would you scale from 10M to 100M metrics/second?

**Answer:**

1. **Linear Scaling**
   ```
   Collectors: 100 → 1,000
   Query services: 50 → 500
   ```

2. **Storage Scaling**
   ```
   Time-Series DB: Scale to 10x nodes
   Storage: 79 PB → 790 PB
   ```

3. **Architecture Changes**
   - Regional deployment (collectors close to services)
   - Better compression (reduce storage 2x)
   - Tiered storage (hot/cold/warm)

---

## Failure Scenarios

### Scenario 1: Time-Series DB Cluster Failure

**Impact:** Metrics cannot be stored, queries fail

**Recovery:**
1. Metrics buffered in Kafka (1-day retention)
2. Restore from backup or rebuild cluster
3. Replay metrics from Kafka
4. System recovers

---

## Level-Specific Expectations

### L4 (Entry Level)

**Q: How does monitoring work?**
- Services emit metrics
- Metrics collected and stored
- Queries retrieve metrics
- Alerts trigger on thresholds

### L5 (Mid Level)

**Q: How do you optimize storage costs?**
- Compression (reduce size 3x)
- Retention policies (hot/cold storage)
- Tiered storage (cheaper for old data)

### L6 (Senior Level)

**Q: How do you design for global scale?**
- Regional deployment
- Multi-region replication
- Adaptive sampling
- Cost optimization strategies

---

## Summary

Key takeaways:
- Time-series databases optimized for metrics
- Pull model (Prometheus) preferred for scalability
- Alert grouping and deduplication reduce fatigue
- Storage costs dominate (optimize aggressively)

