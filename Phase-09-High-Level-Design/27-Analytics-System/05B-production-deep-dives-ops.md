# Analytics System - Production Deep Dives: Operations (Scaling, Monitoring, Security, Simulation, Cost)

## STEP 9: SCALING & RELIABILITY

### Load Balancing Approach

**Ingestion Service Load Balancing:**

**Strategy:** Round-robin with health checks

**Configuration:**
- Health check endpoint: `/health`
- Health check interval: 10 seconds
- Unhealthy threshold: 3 consecutive failures
- Healthy threshold: 1 successful check

**Load Balancer:** AWS Application Load Balancer (ALB) or NGINX

**Session Affinity:** None (stateless service)

**Geographic Distribution:**
- US: 60% of traffic → 3 regions (us-east, us-west, us-central)
- EU: 20% of traffic → 2 regions (eu-west, eu-central)
- Asia: 20% of traffic → 2 regions (ap-southeast, ap-northeast)

**Query Service Load Balancing:**

**Strategy:** Least connections (queries can be long-running)

**Configuration:**
- Health check: `/health` (includes database connectivity)
- Timeout: 5 minutes (for long queries)
- Connection draining: 30 seconds

---

### Rate Limiting Implementation Details

**Distributed Rate Limiting (Redis):**

**Algorithm:** Token bucket

**Implementation:**
```java
// Token bucket per API key
Key: rate_limit:{api_key}
Value: {tokens: 100000, last_refill: timestamp}
TTL: 1 hour

// Refill logic
tokens = min(capacity, tokens + (now - last_refill) * refill_rate)
if (tokens >= 1) {
    tokens--;
    return ALLOW;
} else {
    return DENY;
}
```

**Rate Limit Tiers:**
- Free: 10K events/day
- Standard: 1M events/day
- Enterprise: Unlimited (with SLA)

**Per-Endpoint Limits:**
- Event ingestion: Per API key
- Query API: Per user
- Admin API: Per IP

**Rate Limit Headers:**
```http
X-RateLimit-Limit: 100000
X-RateLimit-Remaining: 99950
X-RateLimit-Reset: 1640000000
```

---

### Circuit Breaker Placement

**Circuit Breakers:**

1. **Ingestion Service → Kafka:**
   - Failure threshold: 50% failures in 1 minute
   - Timeout: 1 second
   - Fallback: Queue events in local buffer, retry later

2. **Query Service → Redis:**
   - Failure threshold: 50% failures in 1 minute
   - Timeout: 100ms
   - Fallback: Query data warehouse directly

3. **Query Service → Data Warehouse:**
   - Failure threshold: 30% failures in 1 minute
   - Timeout: 30 seconds
   - Fallback: Return cached results, queue query

4. **Stream Processor → Kafka:**
   - Failure threshold: 50% failures in 1 minute
   - Timeout: 5 seconds
   - Fallback: Pause processing, alert operators

---

### Timeout and Retry Policies

**Timeout Configuration:**

| Service | Timeout | Rationale |
|---------|---------|-----------|
| Event Ingestion | 1 second | Fast acknowledgment |
| Query Service (Redis) | 100ms | In-memory, should be fast |
| Query Service (Data Warehouse) | 30 seconds | Complex queries take time |
| Stream Processor | 5 seconds | Processing should be fast |

**Retry Policies:**

**Client Retry (Event Ingestion):**
- Strategy: Exponential backoff with jitter
- Max retries: 3
- Backoff: 1s, 2s, 4s
- Jitter: ±20%

**Service Retry (Internal):**
- Strategy: Exponential backoff
- Max retries: 5
- Backoff: 100ms, 200ms, 400ms, 800ms, 1600ms
- Idempotent operations only

---

### Replication and Failover

**PostgreSQL Replication:**

**Configuration:**
- Primary: 1 instance
- Replicas: 2 instances (read replicas)
- Replication: Async streaming replication
- Lag monitoring: < 100ms acceptable

**Failover:**
- Automatic: Patroni or AWS RDS Multi-AZ
- Failover time: < 30 seconds
- Data loss: None (async replication)

**Redis Replication:**

**Configuration:**
- Redis Cluster: 3 masters, 3 replicas
- Replication: Async
- Failover: Automatic (Redis Sentinel)

**Kafka Replication:**

**Configuration:**
- Replication factor: 3
- Min in-sync replicas: 2
- Failover: Automatic (Kafka handles)

---

### Graceful Degradation Strategies

**Under Stress, Drop in This Order:**

1. **Non-Critical Queries:**
   - Ad-hoc analyst queries → Return 503, queue
   - Pre-aggregated queries still work

2. **Real-Time Metrics:**
   - Real-time queries → Fall back to data warehouse (slower)
   - Historical queries still work

3. **Event Ingestion:**
   - Rate limit more aggressively
   - Prioritize high-value API keys
   - Queue events in client SDK

4. **Batch Processing:**
   - Delay batch jobs (not time-critical)
   - Prioritize real-time processing

**What Stays Up:**
- Event ingestion (core function)
- Critical queries (pre-aggregated metrics)
- Real-time dashboards (with degraded freshness)

---

### Disaster Recovery Approach

**Multi-Region Strategy:**

**Primary Region:** US-East (60% of traffic)
**Secondary Regions:** US-West, EU-West, AP-Southeast

**Data Replication:**
- Object Storage: Cross-region replication (3 regions)
- PostgreSQL: Async replication to secondary region
- Kafka: Replicate topics to secondary region

**Failover Procedure:**

1. **Detection:** Automated monitoring detects region failure
2. **DNS Update:** Route traffic to secondary region (< 5 minutes)
3. **Data Sync:** Catch up on missed events (if any)
4. **Verification:** Verify system health
5. **Communication:** Notify stakeholders

**RTO (Recovery Time Objective):** 1 hour
**RPO (Recovery Point Objective):** 5 minutes (max data loss)

**Backup Strategy:**
- **Database Backups**: Daily full backups, hourly incremental
- **Point-in-time Recovery**: Retention period of 7 days
- **Cross-region Replication**: Async replication to DR region
- **Data Warehouse Backups**: Managed service handles backups
- **Kafka Mirroring**: Real-time mirroring to DR region

**Restore Steps:**
1. Detect primary region failure (health checks) - 5 minutes
2. Promote database replica to primary - 10 minutes
3. Update DNS records to point to secondary region - 5 minutes
4. Restore data warehouse from backup if needed - 30 minutes
5. Verify analytics service health and resume traffic - 10 minutes

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Data warehouse replication lag < 5 minutes
- [ ] Kafka replication lag < 5 minutes
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture event ingestion success rate
   - Document active event count
   - Verify data warehouse replication lag (< 5 minutes)
   - Test event ingestion (100 sample events)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+60 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-3 min:** Promote secondary region to primary
   - **T+3-4 min:** Update DNS records (Route53 health checks)
   - **T+4-30 min:** Rebuild data warehouse from Kafka
   - **T+30-40 min:** Warm up cache from data warehouse
   - **T+40-50 min:** Verify all services healthy in DR region
   - **T+50-60 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+60 to T+75 minutes):**
   - Verify RTO < 1 hour: ✅ PASS/FAIL
   - Verify RPO < 5 minutes: Check data warehouse replication lag at failure time
   - Test event ingestion: Ingest 1,000 test events, verify >99% success
   - Test analytics queries: Query 1,000 test events, verify >99% success
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify event integrity: Check events accessible

5. **Data Integrity Verification:**
   - Compare event count: Pre-failover vs post-failover (should match within 5 min)
   - Spot check: Verify 100 random events accessible
   - Check data warehouse: Verify data warehouse intact
   - Test edge cases: High-volume events, complex queries, aggregations

6. **Failback Procedure (T+75 to T+90 minutes):**
   - Restore primary region services
   - Sync data warehouse from DR to primary
   - Verify replication lag < 5 minutes
   - Update DNS to route traffic back to primary
   - Monitor for 15 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 1 hour: Time from failure to service resumption
- ✅ RPO < 5 minutes: Maximum data loss (verified via data warehouse replication lag)
- ✅ Event ingestion works: >99% events ingested successfully
- ✅ No critical events lost: Event count matches pre-failover (within 5 min window)
- ✅ Service resumes within RTO target: All metrics return to baseline

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: Event Ingestion and Real-Time Dashboard

**Step-by-step:**

1. **User Action**: User clicks "Add to Cart" button
2. **Client sends event**: `POST /v1/events` with `{"event_type": "add_to_cart", "user_id": "user123", "product_id": "prod_abc", "timestamp": "..."}`
3. **Analytics Service**: 
   - Validates event
   - Publishes to Kafka: `{"event": {...}, "partition": 3}` (partitioned by user_id)
4. **Real-Time Processor** (consumes Kafka):
   - Receives event from partition 3
   - Aggregates: Updates `add_to_cart_count` for last 5 minutes
   - Updates Redis: `INCR analytics:realtime:add_to_cart:5min`
5. **User views dashboard** (30 seconds later):
   - Request: `GET /v1/dashboards/realtime`
   - Query Redis: `GET analytics:realtime:add_to_cart:5min` → Returns count: 1,234
   - Returns real-time metrics
6. **Response**: `200 OK` with `{"add_to_cart_count": 1234, "time_range": "5min"}`
7. **Result**: Event ingested and visible in dashboard within 1 second

**Total latency: Ingestion ~100ms, Processing ~200ms, Dashboard query ~50ms**

### Journey 2: Batch Analytics and Report Generation

**Step-by-step:**

1. **Background Job** (runs daily at 2 AM):
   - Queries data warehouse: `SELECT event_type, COUNT(*) FROM events WHERE date = '2024-12-19' GROUP BY event_type`
   - Aggregates metrics: Calculates daily totals, averages, trends
   - Stores in analytics DB: `INSERT INTO daily_metrics (date, event_type, count)`
2. **User requests report** (next morning):
   - Request: `GET /v1/reports/daily?date=2024-12-19`
   - Query analytics DB: Returns pre-aggregated metrics
3. **Response**: `200 OK` with report data
4. **Result**: Report generated from pre-aggregated data, < 100ms query time

**Total time: Batch job ~30 minutes, Report query < 100ms**

### Failure & Recovery Walkthrough

**Scenario: Kafka Broker Failure During Event Ingestion**

**RTO (Recovery Time Objective):** < 5 minutes (broker failover + consumer rebalance)  
**RPO (Recovery Point Objective):** < 1 minute (Kafka replication lag)

**Timeline:**

```
T+0s:    Kafka broker 3 crashes (handles partitions 10-14)
T+0-10s: Event ingestion to partitions 10-14 fails
T+10s:   Events buffered in client (local buffer)
T+15s:   Kafka cluster detects broker failure
T+20s:   Partitions 10-14 reassigned to brokers 1, 2, 4, 5
T+25s:   Consumer group rebalances
T+30s:   Event ingestion resumes
T+1min:  Consumers resume from last committed offset
T+5min:  All operations normal
```

**What degrades:**
- Event ingestion delayed for 20-30 seconds
- Events buffered locally (no data loss)
- Real-time dashboards may show stale data

**What stays up:**
- Event ingestion for other partitions
- Batch analytics (reads from data warehouse)
- All read operations

**What recovers automatically:**
- Kafka cluster reassigns partitions
- Consumers rebalance and resume
- No manual intervention required

**What requires human intervention:**
- Investigate root cause of broker failure
- Replace failed broker hardware
- Review Kafka cluster capacity

**Cascading failure prevention:**
- Partition replication prevents data loss
- Client-side buffering prevents event loss
- Circuit breakers prevent retry storms
- Dead letter queue for failed events

**Active-Active vs Active-Passive:**

**We Use: Active-Passive**
- Primary region: Active (handles all traffic)
- Secondary regions: Passive (standby)
- Why: Simpler, lower cost, acceptable RTO

**Alternative: Active-Active**
- Pros: Zero downtime, instant failover
- Cons: Complex, higher cost, data consistency challenges
- Why Not: Not needed for analytics (acceptable downtime)

### Database Connection Pool Configuration

**PostgreSQL Connection Pool (HikariCP):**

```yaml
spring:
  datasource:
    hikari:
      # Pool sizing
      maximum-pool-size: 20          # Max connections per application instance
      minimum-idle: 5                 # Minimum idle connections
      
      # Timeouts
      connection-timeout: 30000       # 30s - Max time to wait for connection from pool
      idle-timeout: 600000            # 10m - Idle connection timeout
      max-lifetime: 1800000           # 30m - Max connection lifetime
      
      # Monitoring
      leak-detection-threshold: 60000 # 60s - Detect connection leaks
      
      # Validation
      connection-test-query: SELECT 1
      validation-timeout: 3000        # 3s - Connection validation timeout
      
      # Register metrics
      register-mbeans: true
```

**Pool Sizing Calculation:**
```
For analytics service:
- 4 CPU cores per pod
- Read-heavy (query operations)
- Calculated: (4 × 2) + 1 = 9 connections minimum
- With safety margin: 20 connections per pod
- 50 pods × 20 = 1,000 max connections to database
- Database max_connections: 1,200 (20% headroom)
```

**Pool Exhaustion Mitigation:**

1. **Monitoring:**
   - Alert when pool usage > 80%
   - Track wait time for connections
   - Monitor connection leak detection

2. **Circuit Breaker:**
   - If pool exhausted for > 5 seconds, open circuit breaker
   - Fail fast instead of blocking

3. **PgBouncer (Connection Pooler):**
   - Place PgBouncer between app and PostgreSQL
   - PgBouncer pool: 100 connections
   - App pools can share PgBouncer connections efficiently

---

### What Breaks First Under Load

**Bottleneck Analysis:**

1. **First Bottleneck: Kafka Throughput**
   - Symptom: Event ingestion starts failing
   - Cause: Kafka cluster can't keep up
   - Mitigation: Add more Kafka brokers, increase partitions

2. **Second Bottleneck: Stream Processor**
   - Symptom: Real-time metrics become stale
   - Cause: Stream processor can't process fast enough
   - Mitigation: Add more Flink workers, optimize processing

3. **Third Bottleneck: Data Warehouse Queries**
   - Symptom: Query timeouts
   - Cause: Too many concurrent queries
   - Mitigation: Query queuing, increase compute

4. **Fourth Bottleneck: Redis Memory**
   - Symptom: Evictions, cache misses
   - Cause: Memory full
   - Mitigation: Increase Redis memory, optimize TTLs

---

## STEP 10: MONITORING & OBSERVABILITY

### Key Metrics

**Golden Signals (SRE):**

1. **Latency:**
   - Event ingestion: p50 < 50ms, p95 < 100ms, p99 < 200ms
   - Query response: p50 < 200ms, p95 < 2s, p99 < 30s
   - Real-time metrics: p50 < 10ms, p95 < 50ms, p99 < 100ms

2. **Error Rate:**
   - Event ingestion: < 0.1% errors
   - Query API: < 1% errors
   - Stream processor: < 0.01% errors

3. **Throughput:**
   - Event ingestion: 1M events/second (peak)
   - Query API: 1K queries/second
   - Stream processor: 1M events/second processed

4. **Saturation:**
   - CPU: < 70% average
   - Memory: < 80% average
   - Disk I/O: < 80% average
   - Network: < 80% of capacity

**Business Metrics:**

- Daily Active Users (DAU)
- Events ingested per day
- Query volume per day
- Revenue metrics (if applicable)

---

### Alerting Thresholds and Escalation

**Alert Levels:**

1. **Critical (Page On-Call):**
   - Event ingestion down (> 1% error rate for 5 minutes)
   - Data warehouse down
   - Kafka cluster down
   - **Response Time:** Immediate

2. **Warning (Ticket):**
   - Query latency > p95 threshold for 10 minutes
   - Stream processor lag > 5 minutes
   - Redis memory > 80%
   - **Response Time:** Within 1 hour

3. **Info (Dashboard):**
   - High query volume
   - Cache hit rate < 80%
   - **Response Time:** Monitor only

**Escalation:**

- **Level 1:** On-call engineer (immediate)
- **Level 2:** Team lead (if not resolved in 30 minutes)
- **Level 3:** Engineering manager (if not resolved in 2 hours)

---

### Logging Strategy

**Structured Logging (JSON):**

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "INFO",
  "service": "ingestion-service",
  "trace_id": "trace_abc123",
  "span_id": "span_xyz789",
  "message": "Event ingested",
  "fields": {
    "event_id": "evt_123",
    "event_type": "page_view",
    "user_id": "user_123",
    "latency_ms": 45
  }
}
```

**Log Levels:**
- ERROR: System errors, failures
- WARN: Degraded performance, retries
- INFO: Normal operations, key events
- DEBUG: Detailed debugging (disabled in production)

**Log Retention:**
- Application logs: 30 days
- Access logs: 90 days
- Audit logs: 1 year

**PII Redaction:**
- User IDs: Hashed (SHA-256)
- IP addresses: Last octet masked
- Email addresses: Redacted
- Custom properties: Configurable redaction rules

---

### Distributed Tracing

**Trace IDs:**
- Generated at API gateway
- Propagated through all services
- Included in logs and metrics

**Span Boundaries:**
- Ingestion Service: One span per event
- Query Service: One span per query
- Stream Processor: One span per batch

**Tracing Backend:** Jaeger or AWS X-Ray

**Sampling:**
- 100% for errors
- 1% for successful requests (to reduce overhead)

---

### Health Checks

**Liveness Probe:**
- Endpoint: `/health/live`
- Checks: Service is running
- Failure: Restart container

**Readiness Probe:**
- Endpoint: `/health/ready`
- Checks: Service can accept requests (database connectivity, Kafka connectivity)
- Failure: Remove from load balancer

**Health Check Example:**
```json
{
  "status": "healthy",
  "checks": {
    "database": "ok",
    "kafka": "ok",
    "redis": "ok"
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

---

### Dashboards

**Golden Signals Dashboard:**
- Latency (p50, p95, p99)
- Error rate
- Throughput
- Saturation (CPU, memory, disk, network)

**Business Metrics Dashboard:**
- Daily Active Users
- Events ingested per day
- Query volume
- Revenue metrics

**Service-Specific Dashboards:**
- Ingestion Service: Events/second, error rate, latency
- Query Service: Queries/second, cache hit rate, query latency
- Stream Processor: Events processed/second, lag, errors

**Tools:** Grafana, Datadog, or CloudWatch

---

## STEP 11: SECURITY CONSIDERATIONS

### Authentication Mechanism

**Event Ingestion:**
- API Key authentication
- API keys stored in database (hashed)
- Rate limits per API key

**Query API:**
- OAuth 2.0 / JWT Bearer tokens
- Tokens issued by identity provider
- Token validation at API gateway

**Admin API:**
- Multi-factor authentication (MFA)
- Role-based access control (RBAC)

---

### Authorization and Access Control

**Role-Based Access Control (RBAC):**

**Roles:**
- Admin: Full access
- Analyst: Read-only access to analytics
- Developer: Read/write access to own project's events
- Viewer: Read-only access to dashboards

**Permissions:**
- Project-level: Users can only access their projects
- Query-level: Users can only query allowed metrics
- Export-level: Users can only export their data

---

### Data Encryption

**At Rest:**
- Object Storage: AES-256 encryption
- PostgreSQL: Encryption at rest (TDE)
- Redis: Encryption at rest (optional, performance impact)

**In Transit:**
- TLS 1.3 for all API communication
- TLS for database connections
- TLS for Kafka communication

**Key Management:**
- AWS KMS or HashiCorp Vault
- Key rotation: Every 90 days
- Audit logging: All key access logged

---

### Input Validation and Sanitization

**Event Validation:**
- Schema validation (JSON Schema)
- Type checking (string, number, boolean)
- Size limits (max 10 KB per event)
- Enum validation (for enum fields)

**Query Validation:**
- SQL injection prevention (parameterized queries)
- Query size limits (max 10 KB query)
- Time range limits (max 1 year)
- Result size limits (max 100 MB)

**Sanitization:**
- Remove null bytes
- Truncate oversized strings
- Escape special characters in logs

---

### Rate Limiting for Abuse Prevention

**Per-User Limits:**
- Free tier: 10K events/day
- Standard tier: 1M events/day
- Enterprise: Unlimited (with SLA)

**Per-IP Limits:**
- 1000 requests/minute per IP
- Prevents DDoS attacks

**Per-API-Key Limits:**
- Based on tier
- Prevents abuse from compromised keys

---

### PII Handling and Compliance

**GDPR Compliance:**
- Right to access: Users can export their data
- Right to deletion: Users can request data deletion
- Data retention: 2 years (configurable)
- Consent tracking: Track user consent for data processing

**Data Deletion:**
- Delete events by userId
- Delete from all storage systems (object storage, data warehouse, Redis)
- Audit log: Track all deletions

**Data Retention:**
- Raw events: 2 years
- Aggregated metrics: 5 years
- Audit logs: 1 year

---

### Security Headers and HTTPS

**Security Headers:**
```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'
```

**HTTPS:**
- TLS 1.3 required
- Certificate: Let's Encrypt or AWS Certificate Manager
- HSTS: Enabled

---

### Threats and Mitigations

**Threats:**

1. **DDoS Attacks:**
   - Mitigation: Rate limiting, CDN (CloudFlare), auto-scaling

2. **SQL Injection:**
   - Mitigation: Parameterized queries, input validation

3. **API Key Theft:**
   - Mitigation: Key rotation, rate limiting, monitoring

4. **Data Exfiltration:**
   - Mitigation: Access controls, audit logging, encryption

5. **Replay Attacks:**
   - Mitigation: Event deduplication (eventId), timestamp validation

---

## STEP 12: SIMULATION (END-TO-END USER JOURNEYS + FAILURE/RECOVERY)

### User Journey 1: Event Ingestion and Real-Time Query

**Scenario:** User clicks "Add to Cart" button, analyst queries revenue metric

**Step-by-Step:**

1. **User Action:**
   - User clicks "Add to Cart" on e-commerce site
   - Client SDK captures event

2. **Event Ingestion:**
   ```
   Client SDK → API Gateway → Ingestion Service
   POST /v1/events
   {
     "eventId": "evt_abc123",
     "eventType": "add_to_cart",
     "userId": "user_123",
     "properties": {"productId": "prod_456", "amount": 29.99}
   }
   ```

3. **Ingestion Service Processing:**
   - Validate API key ✓
   - Validate event schema ✓
   - Check Redis for eventId (dedupe) ✓ (new event)
   - Write to Kafka (topic: events.project_123, partition: 42) ✓
   - Store eventId in Redis (24-hour TTL) ✓
   - Insert metadata into PostgreSQL (async) ✓
   - Return 202 Accepted to client ✓

4. **Stream Processor:**
   - Consumes event from Kafka (partition 42, offset 1,234,567)
   - Processes event: aggregates "add_to_cart" metric
   - Updates Redis: `metric:add_to_cart:1m:2024-01-15T10:30:00Z` = count + 1
   - Commits offset: 1,234,568

5. **Analyst Queries Revenue:**
   ```
   GET /v1/metrics/revenue?startTime=2024-01-15T10:30:00Z&endTime=2024-01-15T10:35:00Z
   ```

6. **Query Service:**
   - Checks Redis for metric aggregates (HIT)
   - Aggregates from Redis time windows
   - Returns response:
   ```json
   {
     "metric": "revenue",
     "data": [
       {"timestamp": "2024-01-15T10:30:00Z", "value": 50000},
       {"timestamp": "2024-01-15T10:31:00Z", "value": 52000}
     ]
   }
   ```

**Latency:**
- Event ingestion: 50ms
- Stream processing: 2 seconds
- Query response: 10ms (from Redis)

---

### User Journey 2: Batch Query on Historical Data

**Scenario:** Analyst queries daily active users for last month

**Step-by-Step:**

1. **Analyst Sends Query:**
   ```
   GET /v1/metrics/daily_active_users?startTime=2024-01-01T00:00:00Z&endTime=2024-01-31T23:59:59Z&granularity=day
   ```

2. **Query Service:**
   - Checks Redis cache (MISS - query not cached)
   - Routes to data warehouse (historical query)

3. **Data Warehouse Query:**
   - Executes SQL query on aggregated metrics table
   - Scans partitions: year=2024, month=01, days 1-31
   - Aggregates daily active users per day
   - Returns 31 rows (one per day)

4. **Query Service:**
   - Caches result in Redis (1-hour TTL)
   - Returns response to analyst

**Latency:**
- Query execution: 2 seconds
- Total response time: 2.1 seconds

---

### Failure & Recovery Walkthrough

**Scenario:** Kafka cluster experiences network partition

**What Happens:**

1. **Detection:**
   - Monitoring detects Kafka cluster partition
   - Alerts triggered (Critical)

2. **Impact:**
   - Event ingestion: Starts failing (can't write to Kafka)
   - Stream processor: Stops processing (can't read from Kafka)
   - Query service: Still works (queries data warehouse)

3. **Degradation:**
   - Ingestion Service: Returns 503 Service Unavailable
   - Client SDK: Queues events locally, retries with backoff
   - Real-time metrics: Stop updating (stale data)
   - Historical queries: Still work (data warehouse unaffected)

4. **Recovery:**
   - Kafka cluster: Automatic failover (within 30 seconds)
   - Ingestion Service: Resumes accepting events
   - Stream Processor: Resumes processing from last committed offset
   - Real-time metrics: Catch up on missed events

5. **Data Loss:**
   - Events queued in client SDK: Not lost (retried)
   - Events in Kafka: Not lost (replicated)
   - Real-time metrics: May be slightly stale (acceptable)

**What Degrades Gracefully:**
- Historical queries (still work)
- Cached query results (still work)
- Event ingestion (queued in client, not lost)

**What Requires Human Intervention:**
- None (automatic recovery)

**Prevention:**
- Circuit breakers prevent cascading failures
- Client SDK queuing prevents event loss
- Monitoring alerts for quick detection

---

## STEP 13: COST ANALYSIS

### Major Cost Drivers

**Storage Costs:**

1. **Object Storage (S3):**
   - 19 PB total storage
   - Hot (30 days): 216 TB × $0.023/GB/month = $4,968/month
   - Warm (1 year): 2.16 PB × $0.012/GB/month = $25,920/month
   - Cold (2+ years): 17 PB × $0.004/GB/month = $68,000/month
   - **Total: ~$99K/month**

2. **Data Warehouse:**
   - 1.6 PB × $5/TB/month = $8,000/month
   - Query compute: $2,000/month
   - **Total: ~$10K/month**

**Compute Costs:**

1. **Ingestion Servers:**
   - 120 servers × $200/month = $24,000/month

2. **Stream Processing:**
   - 126 machines × $500/month = $63,000/month

3. **Batch Processing:**
   - 10 machines × $300/month = $3,000/month

4. **Query Servers:**
   - 20 servers × $200/month = $4,000/month

**Network Costs:**
- Incoming: 2 Gbps × $0.10/GB = $5,400/month
- Outgoing: 2.88 Gbps × $0.10/GB = $7,776/month

**Managed Services:**
- Kafka: $10,000/month
- Redis: $5,000/month
- PostgreSQL: $3,000/month

**Total Monthly Cost: ~$223K/month**

---

### Cost Optimization Strategies

**Current Monthly Cost:** $223,000
**Target Monthly Cost:** $156,100 (30% reduction)

**Top 3 Cost Drivers:**
1. **Data Warehouse (Snowflake):** $150,000/month (67%) - Largest single component
2. **Storage (S3):** $50,000/month (22%) - Event storage
3. **Compute (EC2):** $23,000/month (10%) - Ingestion and query services

**Optimization Strategies (Ranked by Impact):**

1. **Storage Tiering (60% savings on old data):**
   - **Current:** All data in standard storage
   - **Optimization:** 
     - Move old data to cold storage (saves 80% on storage)
     - Use compression (Parquet, Snappy)
   - **Savings:** $30,000/month (60% of $50K storage)
   - **Trade-off:** Slower access to old data (acceptable for archive)

2. **Query Optimization (30% savings on data warehouse):**
   - **Current:** High query load
   - **Optimization:** 
     - Pre-aggregate common queries
     - Cache query results
     - Limit ad-hoc queries
   - **Savings:** $45,000/month (30% of $150K data warehouse)
   - **Trade-off:** Slight complexity increase (acceptable)

3. **Compute Optimization (40% savings):**
   - **Current:** On-demand instances
   - **Optimization:** 
     - Auto-scale based on load
     - Use spot instances for batch processing (60% savings)
     - Right-size instances
   - **Savings:** $9,200/month (40% of $23K compute)
   - **Trade-off:** Possible interruptions for spot instances (acceptable)

4. **Data Retention (20% savings on storage):**
   - **Current:** Long retention periods
   - **Optimization:** 
     - Archive data older than retention period
     - Delete unnecessary data
   - **Savings:** $10,000/month (20% of $50K storage)
   - **Trade-off:** Slight data loss risk (acceptable)

5. **Reserved Capacity (20% savings on data warehouse):**
   - **Current:** On-demand data warehouse
   - **Optimization:** Reserved capacity for baseline
   - **Savings:** $30,000/month (20% of $150K data warehouse)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

**Total Potential Savings:** $124,200/month (56% reduction)
**Optimized Monthly Cost:** $98,800/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Event Ingestion | $0.000223 | $0.000099 | 56% |
| Analytics Query | $0.001 | $0.0004 | 60% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Storage Tiering, Query Optimization → $75K savings
2. **Phase 2 (Month 2):** Compute Optimization, Data Retention → $19.2K savings
3. **Phase 3 (Month 3):** Reserved Capacity → $30K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor event ingestion rates (ensure no degradation)
- Monitor query performance (target < 1s p99)
- Review and adjust quarterly

---

### Build vs Buy Decisions

| Component | Decision | Rationale |
|-----------|----------|-----------|
| Event Ingestion | Build | Custom requirements, high throughput |
| Stream Processing | Buy (Flink) | Complex, use managed service |
| Data Warehouse | Buy (Snowflake) | Petabyte scale, managed service |
| Query Service | Build | Custom query interface |
| Caching | Buy (Redis) | Standard, use managed service |

---

### At What Scale Does This Become Expensive?

**Current Scale:** 1M events/second
**Cost:** ~$223K/month

**10x Scale (10M events/second):**
- Storage: 10x = $990K/month
- Compute: 10x = $940K/month
- **Total: ~$1.9M/month**

**100x Scale (100M events/second):**
- Storage: 100x = $9.9M/month
- Compute: 100x = $9.4M/month
- **Total: ~$19M/month**

**Optimization Needed:**
- At 10x: Aggressive data tiering, query optimization
- At 100x: Custom storage solutions, data sampling

---

### Cost Per User or Per Request Estimate

**Assumptions:**
- 500M users
- 1M events/second = 86.4B events/day
- Events per user per day: 86.4B / 500M = 173 events/user/day

**Cost Per User:**
- Monthly cost: $223K
- Users: 500M
- Cost per user: $223K / 500M = $0.0004/user/month

**Cost Per Event:**
- Monthly cost: $223K
- Events per month: 86.4B × 30 = 2.592T events
- Cost per event: $223K / 2.592T = $0.000086/event

**Cost Per Query:**
- Query cost: $10K/month (data warehouse compute)
- Queries per month: 1K/second × 86,400 × 30 = 2.592M queries
- Cost per query: $10K / 2.592M = $0.004/query

---

## Summary

**Scaling:**
- Horizontal scaling for all services
- Auto-scaling based on load
- Multi-region for disaster recovery

**Monitoring:**
- Golden signals (latency, error rate, throughput, saturation)
- Business metrics
- Distributed tracing

**Security:**
- API key and OAuth authentication
- Encryption at rest and in transit
- GDPR compliance

**Cost:**
- ~$223K/month at 1M events/second
- Optimized through tiering, caching, pre-aggregation


