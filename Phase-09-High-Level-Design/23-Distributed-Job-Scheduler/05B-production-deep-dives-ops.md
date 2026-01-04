# Distributed Job Scheduler - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis.

---

## 1. Scaling & Reliability

### Load Balancing Approach

- Round-robin for scheduler services
- Worker-based job assignment (workers pull jobs)
- Health check-based routing

### Rate Limiting Implementation

**Per-User Rate Limiting:**

```java
@Service
public class JobSubmissionRateLimiter {
    
    public boolean isAllowed(String userId) {
        String key = "rate_limit:job_submission:" + userId;
        // Check rate limit in Redis
        // Limit: 1000 jobs/minute per user
    }
}
```

### Circuit Breaker Placement

| Service | Circuit Breaker | Fallback |
|---------|-----------------|----------|
| Worker Service | Yes | Queue jobs for retry |
| Job Queue Service | Yes | Fallback to database queue |
| Dependency Resolver | Yes | Skip dependency checks |

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| Job Execution | Configurable | 3 | Exponential |
| Worker Heartbeat | 60s | 2 | 10s, 30s |
| Job Scheduling | 5s | 2 | 1s, 2s |

### Replication and Failover

- Multi-AZ deployment (3 availability zones)
- PostgreSQL primary-replica with automatic failover
- Redis cluster with automatic failover
- Worker auto-scaling based on queue depth

### Graceful Degradation

**Priority of features to drop:**

1. **First**: Low-priority jobs (skip priority 8-10)
2. **Second**: Job history queries (can be slower)
3. **Third**: Dependency resolution (execute without dependencies)
4. **Last**: Stop accepting new jobs

### Backup and Recovery

**RPO (Recovery Point Objective):** < 5 minutes
- Maximum acceptable data loss
- Based on database replication lag (5 minutes max)
- Job metadata replicated to read replicas within 5 minutes
- Job state stored in Redis with persistence

**RTO (Recovery Time Objective):** < 15 minutes
- Maximum acceptable downtime
- Time to restore service via failover to secondary region
- Includes database failover, cache warm-up, and traffic rerouting

**Backup Strategy:**
- **Database Backups**: Hourly incremental, daily full
- **Point-in-time Recovery**: Retention period of 7 days
- **Cross-region Replication**: Async replication to DR region
- **Redis Snapshots**: Hourly snapshots of job state

**Restore Steps:**
1. Detect primary region failure (health checks) - 1 minute
2. Promote database replica to primary - 2 minutes
3. Update DNS records to point to secondary region - 1 minute
4. Warm up Redis cache from database - 5 minutes
5. Verify scheduler service health and resume traffic - 6 minutes

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Database replication lag < 5 minutes
- [ ] Kafka replication lag < 5 minutes
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available
- [ ] No critical jobs running during test window (or minimal)

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture job scheduling success rate
   - Document active job count
   - Verify database replication lag (< 5 minutes)
   - Test job scheduling (100 sample jobs)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+15 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-3 min:** Promote secondary region to primary
   - **T+3-4 min:** Update DNS records (Route53 health checks)
   - **T+4-8 min:** Rebalance Kafka partitions
   - **T+8-10 min:** Warm up cache from database
   - **T+10-12 min:** Verify all services healthy in DR region
   - **T+12-15 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+15 to T+25 minutes):**
   - Verify RTO < 15 minutes: ✅ PASS/FAIL
   - Verify RPO < 5 minutes: Check database replication lag at failure time
   - Test job scheduling: Schedule 1,000 test jobs, verify >99% success
   - Test job execution: Verify job execution works correctly
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify job integrity: Check all jobs accessible

5. **Data Integrity Verification:**
   - Compare job count: Pre-failover vs post-failover (should match within 5 min)
   - Spot check: Verify 100 random jobs accessible
   - Check job state: Verify job state matches pre-failover
   - Test edge cases: Long-running jobs, scheduled jobs, failed jobs

6. **Failback Procedure (T+25 to T+30 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag < 5 minutes
   - Update DNS to route traffic back to primary
   - Monitor for 5 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 15 minutes: Time from failure to service resumption
- ✅ RPO < 5 minutes: Maximum data loss (verified via database replication lag)
- ✅ Job scheduling works: >99% jobs scheduled successfully
- ✅ No data loss: Job count matches pre-failover (within 5 min window)
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

### Journey 1: User Submits Scheduled Job

**Step-by-step:**

1. **User Action**: User submits job via `POST /v1/jobs` with `{"name": "daily-report", "schedule": "0 0 * * *", "command": "python generate_report.py"}`
2. **Job Scheduler Service**: 
   - Validates schedule (cron format)
   - Generates job ID: `job_abc123`
   - Creates job record: `INSERT INTO jobs (id, name, schedule, command, status)`
   - Calculates next run time: `2024-12-20 00:00:00`
   - Publishes to Kafka: `{"event": "job_created", "job_id": "job_abc123"}`
3. **Response**: `201 Created` with `{"job_id": "job_abc123", "next_run": "2024-12-20 00:00:00"}`
4. **Scheduler Worker** (consumes Kafka):
   - Adds job to priority queue: `ZADD job_queue 1734652800 job_abc123` (timestamp as score)
   - Monitors queue for jobs due to run
5. **At scheduled time** (2024-12-20 00:00:00):
   - Scheduler worker finds job in queue
   - Assigns to available worker: `PUT /v1/workers/worker_5/assign {"job_id": "job_abc123"}`
   - Worker executes job: Runs `python generate_report.py`
   - Updates status: `UPDATE jobs SET status = 'running', last_run = NOW()`
6. **Job completes** (5 minutes later):
   - Worker sends: `PUT /v1/jobs/job_abc123/complete {"status": "success", "output": "..."}`
   - Updates job: `UPDATE jobs SET status = 'completed', next_run = '2024-12-21 00:00:00'`
7. **Result**: Job scheduled and executed successfully

**Total time: Scheduled execution at 00:00:00, completed in 5 minutes**

### Journey 2: Job Failure and Retry

**Step-by-step:**

1. **Job execution fails** (worker crashes during execution):
   - Worker sends: `PUT /v1/jobs/job_abc123/fail {"error": "Worker crashed"}`
   - Updates job: `UPDATE jobs SET status = 'failed', retry_count = retry_count + 1`
2. **Retry Logic**:
   - Checks retry count: `retry_count = 1 < max_retries = 3`
   - Calculates backoff: `2^retry_count = 2 minutes`
   - Schedules retry: `ZADD job_queue {now + 2min} job_abc123`
3. **Retry executes** (2 minutes later):
   - Worker retries job execution
   - Job succeeds this time
   - Updates status: `UPDATE jobs SET status = 'completed', retry_count = 0`
4. **Result**: Job retried and completed successfully

**Total time: Initial failure + 2 min backoff + retry execution**

### Failure & Recovery Walkthrough

**Scenario: Scheduler Worker Failure**

**RTO (Recovery Time Objective):** < 2 minutes (worker restart + job reassignment)  
**RPO (Recovery Point Objective):** 0 (jobs persisted in database, no data loss)

**Timeline:**

```
T+0s:    Scheduler worker 3 crashes (handles 1000 jobs)
T+0-10s: Job scheduling for affected jobs fails
T+10s:   Health check fails
T+15s:   Kubernetes restarts worker
T+30s:   Worker restarts, loads jobs from database
T+60s:   Worker resumes scheduling from last checkpoint
T+2min:  All jobs rescheduled
```

**What degrades:**
- Job scheduling delayed for 1-2 minutes
- Jobs persisted in database (no data loss)
- Other workers continue normally

**What stays up:**
- Job execution (other workers)
- Job submission
- Job status queries
- All read operations

**What recovers automatically:**
- Kubernetes restarts failed worker
- Worker loads jobs from database
- Jobs rescheduled automatically
- No manual intervention required

**What requires human intervention:**
- Investigate root cause of worker failure
- Review worker resource allocation
- Check for systemic issues

**Cascading failure prevention:**
- Job persistence prevents data loss
- Worker isolation (one crash doesn't affect others)
- Circuit breakers prevent retry storms
- Dead letter queue for failed jobs

### Database Connection Pool Configuration

**Connection Pool Settings (HikariCP):**

```yaml
spring:
  datasource:
    hikari:
      # Pool sizing
      maximum-pool-size: 25          # Max connections per application instance
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
For scheduler service:
- 8 CPU cores per pod
- High transaction rate (job scheduling, status updates)
- Calculated: (8 × 2) + 1 = 17 connections minimum
- With safety margin: 25 connections per pod
- 30 pods × 25 = 750 max connections to database
- Database max_connections: 1,000 (20% headroom)
```

**Redis Connection Pool (Jedis):**

```yaml
redis:
  jedis:
    pool:
      max-active: 30             # Max connections per instance
      max-idle: 15                # Max idle connections
      min-idle: 5                 # Min idle connections
      
    # Timeouts
    connection-timeout: 2000      # 2s - Max time to get connection
    socket-timeout: 3000          # 3s - Max time for operation
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

## 2. Monitoring & Observability

### Key Metrics

**Scheduler Metrics:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Job success rate | > 99% | < 95% | < 90% |
| Job scheduling latency (p99) | < 50ms | > 100ms | > 200ms |
| Worker utilization | > 80% | < 70% | < 60% |
| Queue depth | < 1M | > 5M | > 10M |

### Alerting Thresholds

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| High Job Failure Rate | > 5% for 10 min | Critical | Page on-call |
| Low Worker Utilization | < 70% for 30 min | Warning | Investigate |
| High Queue Depth | > 5M for 10 min | Critical | Scale workers |
| Worker Health Issues | > 10% dead for 5 min | Critical | Page on-call |

---

## 3. Security Considerations

### Authentication and Authorization

- JWT token authentication
- Role-based access control (RBAC)
- Job submission requires auth
- Worker registration requires auth

### Job Isolation

- Jobs run in isolated environments
- Resource limits per job
- Network isolation

### Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **Job Spam** | Rate limiting per user |
| **Resource Exhaustion** | Job timeout, resource limits |
| **Unauthorized Access** | Authentication, authorization |
| **DDoS** | Rate limiting, WAF |

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: Submit and Execute Job

```
1. User submits job
   User → Scheduler Service: Submit job with cron "0 0 * * *"
   Latency: ~50ms

2. Job scheduled
   Scheduler Service → PostgreSQL: INSERT INTO jobs
   Scheduler Service → Redis: ZADD job_queue
   Latency: ~20ms

3. Cron evaluation
   Scheduler Service: Calculate next_run_at
   When time arrives, job added to queue
   Latency: Real-time

4. Worker picks up job
   Worker → Job Queue Service: Get next job
   Job Queue Service → Redis: ZRANGE job_queue 0 0
   Latency: ~5ms

5. Job execution
   Worker: Execute job
   Worker → Scheduler Service: Update job status
   Latency: Job-dependent

6. Job completion
   Worker → Scheduler Service: Job completed
   Scheduler Service → PostgreSQL: UPDATE jobs SET status = 'completed'
   Latency: ~20ms

Total latency: ~95ms (excluding job execution time)
```

### Journey 2: Job with Dependencies

```
1. Job A depends on Job B and Job C
   User → Scheduler Service: Submit Job A with dependencies [B, C]
   │
   ▼
2. Check dependencies
   Dependency Resolver: Check if B and C completed
   If not: Job A remains in queue
   │
   ▼
3. Wait for dependencies
   When B and C complete:
   - Dependency Resolver: Mark Job A as ready
   - Job A added to execution queue
   │
   ▼
4. Execute Job A
   Worker: Execute Job A
   Job A completes
```

---

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | % of Total |
|-----------|--------------|------------|
| Worker Compute | $2,400,000 | 81% |
| Scheduler Service | $4,500 | <1% |
| Database (RDS) | $12,000 | <1% |
| Redis (ElastiCache) | $3,600 | <1% |
| Storage | $5,082 | <1% |
| Network | $50,000 | 2% |
| **Total** | **~$2,970,000** | 100% |

### Cost Optimization Strategies

**Current Monthly Cost:** $2,970,000
**Target Monthly Cost:** $2,079,000 (30% reduction)

**Top 3 Cost Drivers:**
1. **Compute (EC2 Workers):** $2,400,000/month (81%) - Largest single component
2. **Database (RDS):** $400,000/month (13%) - PostgreSQL database
3. **Kafka (MSK):** $100,000/month (3%) - Message queue

**Optimization Strategies (Ranked by Impact):**

1. **Spot Instances (60% savings on workers):**
   - **Current:** On-demand instances for workers
   - **Optimization:** 
     - Spot instances for workers (60% savings)
     - Auto-scaling: Scale workers based on queue depth
   - **Savings:** $1,440,000/month (60% of $2.4M workers)
   - **Trade-off:** Possible interruptions (acceptable, jobs can be retried)

2. **Reserved Instances (40% savings on schedulers):**
   - **Current:** On-demand instances for schedulers
   - **Optimization:** 1-year Reserved Instances for baseline
   - **Savings:** $240,000/month (40% of $600K schedulers)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

3. **Job Batching (20% savings on overhead):**
   - **Current:** Individual job processing
   - **Optimization:** 
     - Batch similar jobs to reduce overhead
     - Reduce per-job processing cost
   - **Savings:** $480,000/month (20% of $2.4M workers)
   - **Trade-off:** Slight latency increase (acceptable)

4. **Database Optimization (25% savings):**
   - **Current:** Over-provisioned database
   - **Optimization:** 
     - Connection pooling
     - Query optimization
   - **Savings:** $100,000/month (25% of $400K database)
   - **Trade-off:** Slight complexity increase (acceptable)

**Total Potential Savings:** $2,260,000/month (76% reduction)
**Optimized Monthly Cost:** $710,000/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Job Execution | $0.00099 | $0.00024 | 76% |
| Job Scheduling | $0.00001 | $0.000002 | 80% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Spot Instances, Reserved Instances → $1.68M savings
2. **Phase 2 (Month 2):** Job Batching, Database Optimization → $580K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor job success rates (ensure no degradation)
- Monitor queue depth (target < 1M pending jobs)
- Review and adjust quarterly

### Cost per Job

```
Monthly jobs: 3 billion
Monthly cost: $2,970,000

Cost per job = $2,970,000 / 3B = $0.00099
= $990 per million jobs
```

---

## Summary

| Aspect | Implementation | Key Metric |
|--------|----------------|------------|
| Load Balancing | Round-robin + worker pull | 15 scheduler instances |
| Rate Limiting | Per-user, per-operation | Redis-based |
| Circuit Breakers | Resilience4j | 50% failure threshold |
| Monitoring | Prometheus + Grafana | > 99% job success |
| Security | JWT + RBAC | Isolated job execution |
| DR | Multi-region | RTO < 15 min |
| Cost | $2.97M/month | $990 per million jobs |

