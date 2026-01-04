# Monitoring & Alerting System - Problem & Requirements

## What is a Monitoring & Alerting System?

A monitoring and alerting system collects metrics from applications and infrastructure, stores them in time-series databases, analyzes them for anomalies, and triggers alerts when thresholds are exceeded. It provides visibility into system health, performance, and behavior.

**Example:**

```
Application emits metrics → Collector → Time-Series DB → Alerting Engine
                                                        ↓
                                                    Alert (if threshold exceeded)
```

### Why Does This Exist?

1. **Proactive Issue Detection**: Detect problems before users notice
2. **Performance Monitoring**: Track latency, throughput, error rates
3. **Capacity Planning**: Understand resource usage trends
4. **SLO Monitoring**: Ensure service level objectives are met
5. **Incident Response**: Quick identification of root causes
6. **Trend Analysis**: Understand system behavior over time

### What Breaks Without It?

- Issues discovered only when users complain
- No visibility into system performance
- Cannot detect anomalies or trends
- Reactive incident response (slow)
- No capacity planning data
- Cannot measure SLOs

---

## Clarifying Questions (Ask the Interviewer)

| Question | Why It Matters | Assumed Answer |
|----------|----------------|----------------|
| What's the scale (metrics/second)? | Determines infrastructure size | 10 million metrics/second |
| How many services need monitoring? | Affects instrumentation | 1,000 services |
| What's the retention period? | Affects storage requirements | 30 days hot, 1 year cold |
| What types of metrics? | Affects storage design | Counters, Gauges, Histograms |
| What alerting channels? | Affects notification system | Email, PagerDuty, Slack |
| What's the target query latency? | Affects storage/indexing | < 100ms p95 for dashboards |
| Do we need anomaly detection? | Affects ML requirements | Yes, basic statistical methods |

---

## Functional Requirements

### Core Features (Must Have)

1. **Metrics Collection**
   - Receive metrics from services (Push and Pull)
   - Support multiple protocols (Prometheus, StatsD, custom)
   - Handle high-volume metric ingestion
   - Validate metric data

2. **Time-Series Storage**
   - Store metrics with timestamps
   - Support efficient time-range queries
   - Index by metric name, labels/tags
   - Support both hot (recent) and cold (historical) storage

3. **Query Interface**
   - Query metrics by name, labels, time range
   - Support aggregation functions (sum, avg, rate, etc.)
   - Support complex queries (PromQL-like)
   - Real-time and historical queries

4. **Alerting Rules**
   - Define alert conditions (thresholds, expressions)
   - Evaluate rules periodically
   - Trigger alerts when conditions are met
   - Support alert grouping and deduplication

5. **Notification System**
   - Send alerts via multiple channels (Email, PagerDuty, Slack)
   - Support alert routing (by service, severity)
   - Alert escalation policies
   - Acknowledge and resolve alerts

6. **Dashboards**
   - Create custom dashboards
   - Visualize metrics (graphs, charts)
   - Real-time updates
   - Share dashboards with teams

### Secondary Features (Nice to Have)

7. **Anomaly Detection**
   - Detect unusual patterns in metrics
   - Machine learning-based detection
   - Reduce false positives

8. **Service Discovery**
   - Automatically discover services
   - Auto-configure monitoring
   - Dynamic alerting rules

9. **Metrics Export**
   - Export metrics to external systems
   - API for integrations
   - Compliance reporting

---

## Non-Functional Requirements

### Performance

| Metric | Target | Rationale |
|--------|--------|-----------|
| Metric ingestion latency | < 10ms p99 | Low overhead on services |
| Query latency (dashboard) | < 100ms p95 | Responsive dashboards |
| Query latency (complex) | < 1s p95 | Acceptable for complex queries |
| Alert evaluation latency | < 5s | Fast alerting |

### Scale

| Metric | Value | Calculation |
|--------|-------|-------------|
| Metrics/second | 10 million | Given assumption |
| Unique metrics | 1 million | 1K metrics per service × 1K services |
| Retention (hot) | 30 days | Fast queries for recent data |
| Retention (cold) | 1 year | Long-term trends and compliance |
| Storage (hot, 30 days) | 260 TB | 10M metrics/s × 16 bytes × 2.59M seconds |
| Storage (cold, 1 year) | 3.1 PB | Additional 335 days |

### Availability

- **99.99% availability**: Critical for production monitoring
- **Data durability**: 99.999999999% (11 nines) - metrics are valuable
- **Graceful degradation**: If storage is full, drop oldest metrics but continue collecting

---

## What's Out of Scope

1. **Log Storage**: Separate logging system handles logs
2. **Tracing**: Separate tracing system handles distributed traces
3. **APM (Application Performance Monitoring)**: Separate APM system
4. **Real-time Streaming Analytics**: Focus on time-series, not streams
5. **Custom Visualization**: Use existing visualization tools (Grafana)
6. **Machine Learning Training**: Basic anomaly detection only

---

## System Constraints

### Technical Constraints

1. **Metric Size**: Average metric is 16 bytes (name + labels + value + timestamp)
2. **Query Performance**: Must support fast queries for dashboards
3. **Retention**: Must balance storage costs with retention needs
4. **Alert Evaluation**: Must evaluate alerts efficiently (millions of rules)

### Business Constraints

1. **Cost**: Storage is expensive, must optimize
2. **Alert Fatigue**: Too many alerts reduce effectiveness
3. **Compliance**: May need long retention for compliance

---

## Success Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Metric ingestion success rate | > 99.99% | Successful metrics / total metrics |
| Query success rate | > 99.9% | Successful queries / total queries |
| Query latency p95 | < 100ms | Time to return query results |
| Alert accuracy | > 95% | True positives / (true positives + false positives) |
| Alert latency | < 30s | Time from condition to alert |
| Dashboard load time | < 2s | Time to render dashboard |

---

## User Stories

### Story 1: Create Alert for High Error Rate

```
As an on-call engineer,
I want to create an alert that triggers when error rate exceeds 1%,
So that I can be notified of production issues quickly.

Acceptance Criteria:
- Define alert rule with metric name and threshold
- Alert triggers within 30 seconds of condition
- Alert sent via PagerDuty
- Alert includes relevant context (service, time range)
```

### Story 2: View Service Dashboard

```
As a service owner,
I want to view a dashboard showing my service's key metrics,
So that I can monitor service health and performance.

Acceptance Criteria:
- Dashboard displays latency, error rate, throughput
- Metrics update in real-time (1-minute refresh)
- Dashboard loads in < 2 seconds
- Supports time range selection (last hour, day, week)
```

### Story 3: Investigate Metric Anomaly

```
As a performance engineer,
I want to query metrics by service and time range,
So that I can investigate performance anomalies.

Acceptance Criteria:
- Query metrics by service name, metric name, time range
- Support aggregation functions (avg, sum, rate)
- Query completes in < 1 second
- Results displayed in graph format
```

---

## Core Components Overview

### 1. Metrics Collector
- Receives metrics from services
- Validates metric data
- Routes metrics to storage
- Supports Push and Pull models

### 2. Time-Series Storage
- Stores metrics with timestamps
- Efficient time-range queries
- Indexes for fast lookup
- Hot and cold storage tiers

### 3. Query Engine
- Executes metric queries
- Supports aggregation functions
- Optimizes query performance
- Caches query results

### 4. Alerting Engine
- Evaluates alert rules
- Triggers alerts when conditions met
- Groups and deduplicates alerts
- Manages alert state

### 5. Notification Service
- Sends alerts via multiple channels
- Routes alerts by rules
- Escalates unresolved alerts
- Tracks alert acknowledgments

### 6. Dashboard Service
- Renders dashboards
- Executes queries for visualizations
- Real-time metric updates
- Dashboard management

---

## Summary

| Aspect | Decision |
|--------|----------|
| Primary use case | Collect, store, query, and alert on metrics for system monitoring |
| Scale | 10M metrics/second, 1M unique metrics |
| Storage | Hot storage (30 days) for fast queries, cold storage (1 year) for compliance |
| Protocols | Prometheus, StatsD, custom |
| Query Language | PromQL-like for flexibility |
| Alerting | Threshold-based with grouping and deduplication |

