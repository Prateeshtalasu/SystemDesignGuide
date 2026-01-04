# Analytics System - Problem & Requirements

## What is an Analytics System?

An Analytics System is a platform that collects, processes, stores, and analyzes user events and behavior data to provide insights, metrics, and business intelligence. It tracks user actions (clicks, page views, purchases, etc.), aggregates them into meaningful metrics, and enables real-time and batch analytics.

**Example Flow:**

```
User clicks "Add to Cart" button
  ↓ Event sent to analytics
Event: {userId: 123, event: "add_to_cart", productId: 456, timestamp: ...}
  ↓ Collected by analytics service
Stored in data warehouse
  ↓ Aggregated into metrics
Dashboard shows: "1,234 add-to-cart events today"
```

### Why Does This Exist?

1. **Product Decisions**: Understand user behavior to improve products
2. **Business Metrics**: Track KPIs like revenue, conversion rates, retention
3. **A/B Testing**: Measure impact of experiments and feature changes
4. **Personalization**: Use behavior data to customize user experience
5. **Fraud Detection**: Identify anomalous patterns
6. **Performance Monitoring**: Track system performance and errors
7. **Compliance**: Audit trails and data retention for regulations

### What Breaks Without It?

- Product teams make decisions based on intuition, not data
- No way to measure impact of new features
- Can't identify which parts of the product need improvement
- Business stakeholders lack visibility into key metrics
- A/B tests can't be evaluated
- No historical data for trend analysis
- Compliance requirements can't be met

---

## Clarifying Questions (Ask the Interviewer)

Before diving into design, a good engineer asks questions to understand scope:

| Question | Why It Matters | Assumed Answer |
|----------|----------------|----------------|
| What's the scale (events per second)? | Determines infrastructure size | 1 million events/second peak |
| What types of events? | Affects schema design | User actions, page views, transactions, errors |
| Real-time vs batch analytics? | Affects architecture | Both: real-time dashboards + batch reports |
| What's the retention period? | Affects storage costs | 2 years for raw events, 5 years for aggregates |
| Do we need A/B testing infrastructure? | Adds complexity | Yes, must support experiment analysis |
| What's the query pattern? | Affects data warehouse design | 80% pre-aggregated queries, 20% ad-hoc |
| Do we need GDPR compliance? | Affects data handling | Yes, must support user data deletion |
| What's the latency requirement for real-time? | Affects streaming architecture | < 5 seconds for real-time dashboards |

---

## Functional Requirements

### Core Features (Must Have)

1. **Event Collection**
   - Accept events from web, mobile, and server-side clients
   - Support multiple event types (page views, clicks, purchases, etc.)
   - Handle high-throughput event ingestion
   - Validate event schema

2. **Event Storage**
   - Store raw events durably
   - Support time-based partitioning
   - Enable efficient querying by time range, user, event type

3. **Real-Time Analytics**
   - Aggregate events in near real-time (< 5 seconds)
   - Support real-time dashboards
   - Track metrics like active users, conversion rates, revenue

4. **Batch Analytics**
   - Process historical data for reports
   - Generate daily/weekly/monthly aggregates
   - Support complex analytical queries

5. **Data Pipeline (ETL)**
   - Extract events from raw storage
   - Transform and enrich events
   - Load into data warehouse
   - Handle schema evolution

6. **Query Interface**
   - SQL-like query interface for analysts
   - Pre-built dashboards for common metrics
   - Export capabilities (CSV, JSON)

### Secondary Features (Nice to Have)

7. **A/B Testing Infrastructure**
   - Track experiment assignments
   - Calculate statistical significance
   - Compare experiment vs control groups

8. **Anomaly Detection**
   - Detect unusual patterns in metrics
   - Alert on significant changes
   - Identify data quality issues

9. **User Segmentation**
   - Segment users by behavior
   - Create custom cohorts
   - Analyze segment-specific metrics

10. **Data Export**
    - Export raw events for external analysis
    - Support bulk data export
    - API for programmatic access

---

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| **Throughput** | 1M events/second peak | Handle traffic spikes during events |
| **Latency (Ingestion)** | < 100ms p99 | Fast event acceptance |
| **Latency (Real-time)** | < 5 seconds | Near real-time dashboards |
| **Latency (Batch)** | Daily reports within 1 hour | Acceptable for historical analysis |
| **Availability** | 99.9% | Analytics is important but not critical path |
| **Durability** | 99.999999999% (11 nines) | Events must not be lost |
| **Scalability** | 10x growth without redesign | Support business growth |
| **Consistency** | Eventual (analytics) | Analytics can tolerate slight delays |

---

## User Stories

### Product Manager

1. **As a product manager**, I want to see daily active users (DAU) in real-time so I can monitor product health.

2. **As a product manager**, I want to analyze conversion funnel so I can identify drop-off points.

3. **As a product manager**, I want to compare metrics before and after a feature launch so I can measure impact.

### Data Analyst

1. **As a data analyst**, I want to query historical events with SQL so I can perform ad-hoc analysis.

2. **As a data analyst**, I want to export data for external tools so I can use specialized analytics software.

3. **As a data analyst**, I want to create custom dashboards so I can track business-specific metrics.

### Engineer

1. **As an engineer**, I want to track application errors and performance so I can debug issues.

2. **As an engineer**, I want to monitor system health metrics so I can ensure reliability.

---

## Constraints & Assumptions

### Constraints

1. **Event Volume**: 1 million events/second peak (86.4 billion events/day)
2. **Storage**: Petabytes of data over time
3. **Schema Evolution**: Events schema changes over time
4. **Compliance**: GDPR requires user data deletion capability
5. **Cost**: Must be cost-effective at scale

### Assumptions

1. Average event size: 500 bytes (JSON)
2. 80% of queries are on pre-aggregated data
3. 20% of queries are ad-hoc on raw events
4. Events arrive from multiple sources (web, mobile, server)
5. Real-time analytics needed for < 5% of use cases
6. Batch analytics sufficient for most reporting needs
7. Data retention: 2 years raw, 5 years aggregates

---

## Out of Scope

1. **Machine Learning Models**: Training ML models on analytics data (separate system)
2. **Real-time Personalization**: Using analytics for real-time recommendations (separate system)
3. **Data Visualization**: Building custom visualization tools (use existing BI tools)
4. **Data Science Notebooks**: Jupyter-style interactive analysis (separate tooling)
5. **Stream Processing**: Complex stream processing beyond aggregation (use specialized tools)

---

## Key Challenges

1. **High Throughput**: Ingesting millions of events per second
2. **Schema Evolution**: Handling schema changes without breaking existing queries
3. **Data Quality**: Ensuring events are valid and complete
4. **Cost Management**: Storing petabytes cost-effectively
5. **Query Performance**: Fast queries on massive datasets
6. **Real-time vs Batch**: Balancing real-time needs with batch efficiency
7. **Compliance**: Supporting GDPR deletion requests efficiently

---

## Success Metrics (SLOs)

### System Metrics

- **Event Ingestion Success Rate**: > 99.9% (events successfully stored)
- **Query Latency (p95)**: < 2 seconds for pre-aggregated queries
- **Query Latency (p95)**: < 30 seconds for ad-hoc queries
- **Data Freshness**: Real-time metrics updated within 5 seconds
- **System Availability**: 99.9% uptime

### Business Metrics

- **Daily Active Users (DAU)**: Tracked accurately
- **Conversion Rate**: Measured with < 1% error
- **Revenue Metrics**: Accurate to the dollar
- **A/B Test Results**: Available within 1 hour of experiment end

---

## Technology Stack Considerations

### Event Collection
- **Client SDKs**: JavaScript, iOS, Android, server-side libraries
- **Ingestion Service**: High-throughput API gateway
- **Message Queue**: Kafka for buffering and decoupling

### Storage
- **Raw Events**: Object storage (S3) or time-series DB
- **Data Warehouse**: Snowflake, BigQuery, or Redshift
- **Real-time Store**: Redis or in-memory database

### Processing
- **Stream Processing**: Kafka Streams, Flink, or Spark Streaming
- **Batch Processing**: Spark, Hadoop, or cloud data processing
- **ETL**: Airflow, Luigi, or cloud ETL tools

### Query & Visualization
- **Query Engine**: Presto, Trino, or data warehouse SQL
- **BI Tools**: Tableau, Looker, or custom dashboards

---

## Design Priorities

1. **Reliability First**: Events must not be lost
2. **Scalability**: Handle 10x growth
3. **Cost Efficiency**: Optimize storage and compute costs
4. **Query Performance**: Fast analytics for business decisions
5. **Flexibility**: Support various event types and schemas

---

## Next Steps

After requirements are clear, we'll design:
1. Capacity estimation (events, storage, bandwidth)
2. API design for event ingestion and queries
3. Data model and storage architecture
4. Real-time and batch processing pipelines
5. Query interface and dashboards


