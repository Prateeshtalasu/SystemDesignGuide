# Centralized Logging System - Problem & Requirements

## What is a Centralized Logging System?

A centralized logging system collects logs from distributed services and infrastructure, stores them in a searchable format, and provides querying capabilities for debugging, auditing, and analysis. It aggregates logs from multiple sources into a single searchable repository.

**Example:**

```
Service A logs → Collector → Centralized Storage → Search Interface
Service B logs → Collector →                    → User queries logs
Service C logs → Collector →                    →
```

### Why Does This Exist?

1. **Centralized Debugging**: Search logs from all services in one place
2. **Distributed Tracing**: Correlate logs across services using trace IDs
3. **Compliance**: Audit logs for security and compliance requirements
4. **Performance Analysis**: Identify performance issues from log patterns
5. **Incident Response**: Quick log analysis during incidents
6. **Trend Analysis**: Understand system behavior over time

### What Breaks Without It?

- Must SSH into each server to view logs
- Cannot correlate logs across services
- No search capability across all logs
- Difficult to debug distributed issues
- No audit trail for compliance
- Reactive incident response (slow)

---

## Clarifying Questions (Ask the Interviewer)

| Question | Why It Matters | Assumed Answer |
|----------|----------------|----------------|
| What's the scale (logs/second)? | Determines infrastructure size | 100 million log lines/second |
| How many services? | Affects log sources | 1,000 services |
| What's the retention period? | Affects storage requirements | 7 days hot, 90 days cold |
| What log formats? | Affects parsing | JSON, plain text, syslog |
| What's the target query latency? | Affects storage/indexing | < 1s p95 for search |
| Do we need real-time streaming? | Affects architecture | Yes, near real-time (< 1 minute delay) |
| What search capabilities? | Affects indexing | Full-text search, field filtering, time-range queries |

---

## Functional Requirements

### Core Features (Must Have)

1. **Log Collection**
   - Receive logs from services (Push and Pull)
   - Support multiple protocols (HTTP, syslog, file-based)
   - Handle high-volume log ingestion
   - Parse structured and unstructured logs

2. **Log Storage**
   - Store logs with timestamps
   - Support efficient time-range queries
   - Index by service, log level, fields
   - Support both hot (recent) and cold (historical) storage

3. **Log Search**
   - Full-text search across logs
   - Filter by service, log level, time range
   - Field-based queries (JSON fields)
   - Complex queries (boolean, regex)

4. **Real-time Streaming**
   - Stream logs in real-time (tail -f like)
   - Near real-time ingestion (< 1 minute delay)
   - Live log viewing

5. **Log Retention**
   - Configurable retention policies
   - Automatic archival to cold storage
   - Automatic deletion after retention period

### Secondary Features (Nice to Have)

6. **Log Aggregation**
   - Aggregate log patterns
   - Error rate analysis
   - Trend analysis

7. **Alerting**
   - Alert on log patterns (error spikes)
   - Integration with monitoring system

8. **Log Parsing**
   - Automatic log format detection
   - Custom parsers for different formats
   - Field extraction

---

## Non-Functional Requirements

### Performance

| Metric | Target | Rationale |
|--------|--------|-----------|
| Log ingestion latency | < 10ms p99 | Low overhead on services |
| Search latency | < 1s p95 | Fast debugging |
| Search latency | < 5s p99 | Acceptable for complex queries |
| Real-time delay | < 1 minute | Near real-time streaming |

### Scale

| Metric | Value | Calculation |
|--------|-------|-------------|
| Log lines/second | 100 million | Given assumption |
| Average log size | 500 bytes | Typical log line |
| Retention (hot) | 7 days | Fast queries for recent logs |
| Retention (cold) | 90 days | Compliance and analysis |
| Storage (hot, 7 days) | 3.02 PB | 100M logs/s × 500 bytes × 604,800s |
| Storage (cold, 90 days) | 38.9 PB | Additional 83 days |

### Availability

- **99.99% availability**: Critical for debugging and compliance
- **Data durability**: 99.999999999% (11 nines) - logs are valuable
- **Graceful degradation**: If storage is full, drop oldest logs but continue collecting

---

## What's Out of Scope

1. **Metrics Storage**: Separate metrics system handles metrics
2. **Tracing**: Separate tracing system handles distributed traces
3. **Log Analysis ML**: No machine learning for log analysis (future feature)
4. **Log Transformation**: Basic parsing only, no complex transformations
5. **Multi-tenancy**: Single-tenant system (internal use)

---

## System Constraints

### Technical Constraints

1. **Log Size**: Average log line is 500 bytes, maximum 64 KB
2. **Query Performance**: Must support fast queries for debugging
3. **Retention**: Must balance storage costs with retention needs
4. **Real-time**: Near real-time ingestion required

### Business Constraints

1. **Cost**: Storage is expensive, must optimize
2. **Compliance**: May need long retention for compliance
3. **Performance Impact**: Logging must not slow down services significantly

---

## Success Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Log ingestion success rate | > 99.99% | Successful logs / total logs |
| Search success rate | > 99.9% | Successful searches / total searches |
| Search latency p95 | < 1s | Time to return search results |
| Real-time delay | < 1 minute | Time from log generation to searchable |
| Storage efficiency | < 4 PB for 7 days | Actual storage used |

---

## User Stories

### Story 1: Search Logs for Error

```
As an on-call engineer,
I want to search for error logs by service and time range,
So that I can quickly find the root cause of a production incident.

Acceptance Criteria:
- Search logs by service name, log level, time range
- Filter by keywords, fields
- View log context (surrounding logs)
- Complete search in < 5 seconds
```

### Story 2: Stream Logs in Real-time

```
As a developer,
I want to stream logs from my service in real-time,
So that I can monitor service behavior during development.

Acceptance Criteria:
- Stream logs with < 1 minute delay
- Filter by service, log level
- View logs as they arrive
- Support tail-like interface
```

### Story 3: Analyze Error Patterns

```
As a performance engineer,
I want to analyze error patterns over time,
So that I can identify recurring issues.

Acceptance Criteria:
- Aggregate errors by service, error type
- Time-series visualization
- Compare error rates across time ranges
- Export data for analysis
```

---

## Core Components Overview

### 1. Log Collector
- Receives logs from services
- Parses log formats
- Routes logs to storage
- Supports Push and Pull models

### 2. Log Storage
- Stores logs with timestamps
- Efficient time-range queries
- Indexes for fast search
- Hot and cold storage tiers

### 3. Search Engine
- Executes log queries
- Full-text search
- Field-based filtering
- Aggregation capabilities

### 4. Real-time Streamer
- Streams logs in real-time
- Near real-time ingestion
- Live log viewing

---

## Summary

| Aspect | Decision |
|--------|----------|
| Primary use case | Collect, store, search, and stream logs for debugging and compliance |
| Scale | 100M log lines/second, 1,000 services |
| Storage | Hot storage (7 days) for fast queries, cold storage (90 days) for compliance |
| Formats | JSON, plain text, syslog |
| Search | Full-text search, field filtering, time-range queries |
| Real-time | Near real-time streaming (< 1 minute delay) |

