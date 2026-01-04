# Distributed Tracing System - Problem & Requirements

## What is Distributed Tracing?

Distributed tracing is a method used to profile and monitor applications, especially those built using a microservices architecture. It tracks requests as they flow through multiple services, providing a complete view of the request journey across distributed systems.

**Example:**

```
User makes request → API Gateway → Order Service → Inventory Service
                                         ↓
                                   Payment Service → Database
```

A distributed tracing system captures timing information and metadata for each service call, creating a complete trace that shows where time is spent and where errors occur.

### Why Does This Exist?

1. **Debugging Complex Systems**: In microservices, a request goes through multiple services. Without tracing, finding where a problem occurs is like searching for a needle in a haystack.

2. **Performance Analysis**: Understand latency at each service level, identify bottlenecks, and optimize the critical path.

3. **Error Tracking**: Quickly identify which service failed and in what context.

4. **Service Dependencies**: Visualize service call graphs and understand system architecture.

5. **SLO Monitoring**: Track request latencies and error rates across service boundaries.

### What Breaks Without It?

- Debugging takes hours or days instead of minutes
- No visibility into cross-service request flows
- Cannot measure end-to-end latency accurately
- Hard to identify slow services in a chain
- Difficult to understand service dependencies
- Cannot correlate errors across services

---

## Clarifying Questions (Ask the Interviewer)

Before diving into design, a good engineer asks questions to understand scope:

| Question | Why It Matters | Assumed Answer |
|----------|----------------|----------------|
| What's the scale (requests/second)? | Determines infrastructure size | 1 million requests/second |
| How many services need tracing? | Affects instrumentation complexity | 500 microservices |
| What's the retention period? | Affects storage requirements | 7 days hot, 90 days cold |
| What sampling rate is acceptable? | Affects storage and cost | 1% sampling for normal traffic, 100% for errors |
| Do we need real-time analysis? | Affects processing architecture | Near real-time (< 1 minute delay) |
| What protocols to support? | Affects collector design | OpenTelemetry, Jaeger, Zipkin |
| What's the target latency overhead? | Affects instrumentation design | < 1% overhead per span |
| Do we need trace visualization UI? | Affects frontend requirements | Yes, web-based UI |

---

## Functional Requirements

### Core Features (Must Have)

1. **Span Collection**
   - Receive spans from instrumented services
   - Support multiple protocols (OpenTelemetry, Jaeger, Zipkin)
   - Handle high-volume span ingestion
   - Validate span data

2. **Trace Storage**
   - Store complete traces (all spans for a trace ID)
   - Support time-based queries
   - Index by trace ID, service name, operation name
   - Support both hot (recent) and cold (historical) storage

3. **Trace Retrieval**
   - Query traces by trace ID
   - Search traces by service, operation, tags
   - Filter by time range
   - Retrieve full trace with all spans

4. **Trace Visualization**
   - Display trace timeline (waterfall view)
   - Show service call graph
   - Highlight errors and slow spans
   - Display span metadata and logs

5. **Sampling**
   - Head-based sampling (at trace start)
   - Tail-based sampling (after trace completes)
   - Adaptive sampling based on load
   - Error sampling (always sample errors)

### Secondary Features (Nice to Have)

6. **Service Dependency Analysis**
   - Generate service dependency graph
   - Identify critical paths
   - Detect circular dependencies

7. **Performance Analytics**
   - P50, P95, P99 latency per service
   - Error rates per service
   - Throughput metrics
   - Comparison across time ranges

8. **Alerting**
   - Alert on high error rates
   - Alert on latency spikes
   - Alert on missing traces

9. **Trace Correlation**
   - Correlate traces with logs
   - Correlate traces with metrics
   - Link traces to incidents

---

## Non-Functional Requirements

### Performance

| Metric | Target | Rationale |
|--------|--------|-----------|
| Span ingestion latency | < 10ms p99 | Low overhead on services |
| Trace query latency | < 500ms p95 | Fast debugging |
| Trace query latency | < 2s p99 | Acceptable for deep queries |
| Sampling overhead | < 1% per span | Minimal performance impact |
| UI load time | < 2 seconds | Good user experience |

### Scale

| Metric | Value | Calculation |
|--------|-------|-------------|
| Requests/second | 1 million | Given assumption |
| Spans per request | 50 (average) | Multi-service architecture |
| Spans/second (raw) | 50 million | 1M req/s × 50 spans |
| Sampling rate | 1% | Cost optimization |
| Spans/second (sampled) | 500,000 | 50M × 0.01 |
| Traces/second | 10,000 | 1M req/s × 0.01 sampling |
| Storage (hot, 7 days) | 3 PB | 500K spans/s × 2KB × 604,800s |
| Storage (cold, 90 days) | 38 PB | Additional 83 days |

### Availability

- **99.9% availability**: System must be highly available for production debugging
- **Data durability**: 99.999999999% (11 nines) - traces are valuable for debugging
- **Graceful degradation**: If storage is full, drop oldest traces but continue collecting

### Latency Targets

- **Span ingestion**: p99 < 10ms (should not slow down services)
- **Trace queries**: p95 < 500ms, p99 < 2s
- **UI rendering**: < 2 seconds for trace visualization

---

## What's Out of Scope

To keep the design focused, we explicitly exclude:

1. **Log Storage**: Separate logging system handles logs (can correlate via trace ID)
2. **Metrics Storage**: Separate metrics system handles metrics
3. **Service Instrumentation**: Services are already instrumented (we just collect)
4. **Real-time Alerting**: Basic alerting only, advanced alerting is separate system
5. **Trace Analytics ML**: No machine learning for anomaly detection (future feature)
6. **Multi-tenancy**: Single-tenant system (internal use)
7. **Trace Replay**: Cannot replay requests, only view traces

---

## System Constraints

### Technical Constraints

1. **Span Size**: Average span is 2 KB, maximum 64 KB
2. **Trace Size**: Average trace has 50 spans, maximum 10,000 spans
3. **Network**: Services send spans over HTTP/gRPC to collectors
4. **Storage**: Must use distributed storage for scale
5. **Query Performance**: Must support fast trace lookups by ID

### Business Constraints

1. **Cost**: Storage is expensive, sampling is critical
2. **Performance Impact**: Tracing must not slow down services significantly
3. **Compliance**: May need to redact PII from traces
4. **Retention**: Legal/compliance may require longer retention

---

## Success Metrics

How do we know if the tracing system is working well?

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Span ingestion success rate | > 99.9% | Successful spans / total spans |
| Trace query success rate | > 99% | Successful queries / total queries |
| Span ingestion latency p99 | < 10ms | Time from service to storage |
| Trace query latency p95 | < 500ms | Time to retrieve and display trace |
| Storage efficiency | < 3 PB for 7 days | Actual storage used |
| Sampling accuracy | Within 1% of target | Actual sampling rate vs target |
| UI availability | > 99.9% | Uptime of visualization UI |

---

## User Stories

### Story 1: Debug Production Incident

```
As an on-call engineer,
I want to search for traces by error tag and time range,
So that I can quickly find the root cause of a production incident.

Acceptance Criteria:
- Search traces by error tag within last hour
- View complete trace with all spans
- See error details and stack traces
- Identify which service first failed
- Complete search in < 5 seconds
```

### Story 2: Performance Analysis

```
As a performance engineer,
I want to view latency percentiles per service over time,
So that I can identify performance regressions.

Acceptance Criteria:
- Query latency metrics by service
- Compare across time ranges
- Identify services with latency spikes
- View service dependency graph
- Export data for further analysis
```

### Story 3: Trace Sampling Configuration

```
As a platform engineer,
I want to configure sampling rates per service,
So that I can balance cost and observability.

Acceptance Criteria:
- Set sampling rate per service
- Enable 100% sampling for error traces
- Adjust sampling dynamically
- Monitor sampling overhead
- View actual vs target sampling rates
```

---

## Core Components Overview

### 1. Span Collector
- Receives spans from services
- Validates span data
- Routes spans to storage
- Handles protocol translation

### 2. Trace Aggregator
- Groups spans by trace ID
- Reconstructs complete traces
- Handles out-of-order spans
- Manages trace completeness

### 3. Storage Layer
- Hot storage (recent traces, fast access)
- Cold storage (historical traces, cheaper)
- Indexes for fast lookup
- Time-based partitioning

### 4. Query Service
- Search traces by ID
- Search by service/operation/tags
- Filter by time range
- Aggregate trace metrics

### 5. Visualization UI
- Display trace timeline
- Show service graph
- Highlight errors
- Display metadata

### 6. Sampling Service
- Head-based sampling decisions
- Tail-based sampling buffering
- Adaptive sampling adjustments
- Error sampling (always sample)

---

## Interview Tips

### What Interviewers Look For

1. **Sampling understanding**: Do you understand why sampling is critical at scale?
2. **Storage design**: Can you handle petabytes of data efficiently?
3. **Query performance**: How do you index and query traces quickly?
4. **Out-of-order handling**: Spans may arrive out of order, how do you handle this?

### Common Mistakes

1. **No sampling strategy**: Trying to store all spans is cost-prohibitive
2. **Naive storage**: Using simple databases that don't scale
3. **Synchronous processing**: Blocking services on span ingestion
4. **Ignoring out-of-order spans**: Spans may arrive out of order due to network delays

### Good Follow-up Questions from Interviewer

- "How do you handle spans arriving out of order?"
- "What happens when a service doesn't propagate trace context?"
- "How do you decide which traces to sample?"
- "How do you handle trace IDs that are too large?"

---

## Summary

| Aspect | Decision |
|--------|----------|
| Primary use case | Collect, store, and visualize distributed traces for debugging and performance analysis |
| Scale | 1M requests/second, 500K spans/second after sampling |
| Sampling | 1% head-based sampling, 100% error sampling, tail-based sampling for important traces |
| Storage | Hot storage (7 days) for fast access, cold storage (90 days) for compliance |
| Protocols | OpenTelemetry, Jaeger, Zipkin support |
| Latency overhead | < 1% per span, < 10ms p99 ingestion latency |


