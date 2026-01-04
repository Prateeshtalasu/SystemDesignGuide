# Distributed Configuration Management - Production Deep Dives: Operations

## STEP 9: SCALING & RELIABILITY

**Load Balancing:** Round-robin with health checks
**Rate Limiting:** Token bucket per user (10K/hour)
**Circuit Breakers:** Fail fast on database/Redis failures
**Replication:** PostgreSQL (primary + 2 replicas), Redis (3 masters, 3 replicas)
**Failover:** Automatic (< 30 seconds)

## STEP 10: MONITORING & OBSERVABILITY

**Key Metrics:**
- Read latency: p99 < 10ms
- Write latency: p99 < 100ms
- Update propagation: < 5 seconds to 99% of services
- Cache hit rate: > 80%
- Error rate: < 0.1%

**Alerting:**
- Critical: Service down, high error rate
- Warning: High latency, low cache hit rate

**Logging:**
- Structured JSON logs
- Audit trail for all changes
- 1-year retention for audit logs

## STEP 11: SECURITY

**Authentication:** OAuth 2.0 / JWT
**Authorization:** RBAC (Admin, Developer, Viewer)
**Encryption:** TLS in transit, encryption at rest for secrets
**Secrets:** Stored in Vault, encrypted
**Audit:** All changes logged with user, timestamp, change log

## STEP 12: SIMULATION

**User Journey: Configuration Update**

1. Admin updates database URL
2. Configuration Service writes to PostgreSQL (version 43)
3. Cache invalidated (Redis key deleted)
4. Update published to Kafka
5. Update Propagation Service consumes from Kafka
6. WebSocket clients notified (< 5 seconds)
7. Services reload configuration
8. Update complete

**Failure Scenario: PostgreSQL Primary Fails**

- Automatic failover to replica (< 30 seconds)
- Writes resume
- No data loss (async replication)
- Cache unaffected

## STEP 13: COST ANALYSIS

**Storage:** ~$0.50/month (4.2 GB)
**Compute:** ~$14K/month (80 servers)
**Total:** ~$14K/month

### Cost Optimization Strategies

**Current Monthly Cost:** $14,000
**Target Monthly Cost:** $9,800 (30% reduction)

**Top 3 Cost Drivers:**
1. **Compute (EC2):** $12,000/month (86%) - Largest single component
2. **Database (RDS):** $1,500/month (11%) - PostgreSQL database
3. **Storage (S3):** $500/month (4%) - Configuration storage

**Optimization Strategies (Ranked by Impact):**

1. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand instances
   - **Optimization:** 1-year Reserved Instances for 80% of compute
   - **Savings:** $4,800/month (40% of $12K compute)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

2. **Cache Optimization (30% savings on compute):**
   - **Current:** High cache miss rate
   - **Optimization:** 
     - Optimize cache TTL
     - Increase cache hit rate to 95%
   - **Savings:** $3,600/month (30% of $12K compute)
   - **Trade-off:** Slight memory increase (acceptable)

3. **Right-sizing (20% savings on compute):**
   - **Current:** Over-provisioned instances
   - **Optimization:** 
     - Use smaller instances for non-critical services
     - Monitor actual usage
   - **Savings:** $2,400/month (20% of $12K compute)
   - **Trade-off:** Need careful monitoring to avoid performance issues

4. **Archive Old Versions (50% savings on storage):**
   - **Current:** All versions in standard storage
   - **Optimization:** 
     - Archive old versions to S3 Glacier (80% cheaper)
     - Keep recent versions in standard storage
   - **Savings:** $250/month (50% of $500 storage)
   - **Trade-off:** Slower access to old versions (acceptable)

**Total Potential Savings:** $11,050/month (79% reduction)
**Optimized Monthly Cost:** $2,950/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Configuration Read | $0.000001 | $0.0000002 | 80% |
| Configuration Write | $0.00001 | $0.000002 | 80% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Reserved Instances, Cache Optimization → $8.4K savings
2. **Phase 2 (Month 2):** Right-sizing, Archive Old Versions → $2.65K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor configuration read/write latency (ensure no degradation)
- Monitor cache hit rates (target >95%)
- Review and adjust quarterly

**Optimization:**
- Use smaller instances for non-critical services
- Optimize cache TTL
- Archive old versions

---

## Backup and Recovery

**RPO (Recovery Point Objective):** < 1 minute
- Maximum acceptable data loss
- Based on database replication lag (1 minute max)
- Configuration changes replicated to read replicas within 1 minute
- Critical for configuration consistency

**RTO (Recovery Time Objective):** < 5 minutes
- Maximum acceptable downtime
- Time to restore service via failover to secondary region
- Includes database failover, cache warm-up, and traffic rerouting

**Backup Strategy:**
- **Database Backups**: Daily full backups, hourly incremental
- **Point-in-time Recovery**: Retention period of 7 days
- **Cross-region Replication**: Async replication to DR region
- **Redis Snapshots**: Hourly snapshots of cached configurations
- **Configuration Version History**: All versions retained for audit

**Restore Steps:**
1. Detect primary region failure (health checks) - 30 seconds
2. Promote database replica to primary - 1 minute
3. Update DNS records to point to secondary region - 30 seconds
4. Warm up Redis cache from database - 2 minutes
5. Verify configuration service health and resume traffic - 1 minute

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Database replication lag < 1 minute
- [ ] Redis replication lag < 1 minute
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture configuration read/write success rate
   - Document active configuration count
   - Verify database replication lag (< 1 minute)
   - Test configuration operations (100 sample operations)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+5 minutes):**
   - **T+0-30s:** Detect failure via health checks
   - **T+30s-1min:** Promote database replica to primary
   - **T+1-2min:** Update DNS records (Route53 health checks)
   - **T+2-3min:** Warm up Redis cache from database
   - **T+3-4min:** Verify all services healthy in DR region
   - **T+4-5min:** Resume traffic to DR region

4. **Post-Failover Validation (T+5 to T+10 minutes):**
   - Verify RTO < 5 minutes: ✅ PASS/FAIL
   - Verify RPO < 1 minute: Check database replication lag at failure time
   - Test configuration read: Read 1,000 test configurations, verify 100% success
   - Test configuration write: Write 100 test configurations, verify 100% success
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify configuration propagation: Check configurations propagate correctly

5. **Data Integrity Verification:**
   - Compare configuration count: Pre-failover vs post-failover (should match within 1 min)
   - Spot check: Verify 100 random configurations accessible
   - Check version history: Verify version history intact
   - Test edge cases: Large configurations, concurrent updates, version conflicts

6. **Failback Procedure (T+10 to T+12 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag < 1 minute
   - Update DNS to route traffic back to primary
   - Monitor for 2 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 5 minutes: Time from failure to service resumption
- ✅ RPO < 1 minute: Maximum data loss (verified via database replication lag)
- ✅ Configuration operations work: 100% operations successful
- ✅ No data loss: Configuration count matches pre-failover (within 1 min window)
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

### Journey 1: Configuration Update and Propagation

**Step-by-step:**

1. **Admin updates config**: Admin updates config via `PUT /v1/configs/database.url` with `{"value": "postgresql://new-db:5432/app", "version": "v2"}`
2. **Config Service**: 
   - Validates config
   - Creates new version: `INSERT INTO configs (key, value, version, created_at)`
   - Publishes to Kafka: `{"event": "config_updated", "key": "database.url", "version": "v2"}`
3. **Response**: `200 OK` with `{"key": "database.url", "version": "v2", "updated_at": "..."}`
4. **Config Watcher** (consumes Kafka):
   - Receives update event
   - Invalidates cache: `DEL config:database.url` (Redis)
   - Notifies subscribed services via WebSocket: `{"type": "config_update", "key": "database.url", "version": "v2"}`
5. **Service receives notification** (user-service):
   - Receives WebSocket message
   - Fetches new config: `GET /v1/configs/database.url` → Returns v2
   - Updates local cache
   - Reloads configuration
6. **Result**: Config updated and propagated to all services within 2 seconds

**Total latency: Update ~100ms, Notification ~500ms, Service reload ~1s**

### Journey 2: Configuration Rollback

**Step-by-step:**

1. **Admin rolls back**: Admin rolls back config via `POST /v1/configs/database.url/rollback` with `{"version": "v1"}`
2. **Config Service**: 
   - Fetches previous version: `SELECT * FROM configs WHERE key = 'database.url' AND version = 'v1'`
   - Creates new version with old value: `INSERT INTO configs (key, value, version) VALUES ('database.url', 'postgresql://old-db:5432/app', 'v3')`
   - Publishes to Kafka: `{"event": "config_rolled_back", "key": "database.url", "from_version": "v2", "to_version": "v3"}`
3. **Config Watcher**:
   - Notifies services: Config rolled back to v1 value
4. **Services reload**: All services reload with previous configuration
5. **Result**: Config rolled back and propagated within 2 seconds

**Total latency: Rollback ~100ms, Notification ~500ms, Service reload ~1s**

### Failure & Recovery Walkthrough

**Scenario: Config Service Failure During Update**

**RTO (Recovery Time Objective):** < 5 minutes (service restart + cache recovery)  
**RPO (Recovery Point Objective):** 0 (configs persisted in database, no data loss)

**Timeline:**

```
T+0s:    Config service crashes (during config update)
T+0-5s:  Config updates fail
T+5s:    Health check fails
T+10s:   Kubernetes restarts service
T+15s:   Service restarts, loads configs from database
T+20s:   Cache warmed from database
T+30s:   Config updates resume
T+5min:  All operations normal
```

**What degrades:**
- Config updates fail for 15-20 seconds
- Configs persisted in database (no data loss)
- Config reads may hit database (slightly higher latency)

**What stays up:**
- Config reads (from database or cache)
- Service operations (using cached configs)
- All read operations

**What recovers automatically:**
- Kubernetes restarts failed service
- Service loads configs from database
- Cache warmed automatically
- No manual intervention required

**What requires human intervention:**
- Investigate root cause of service failure
- Review service resource allocation
- Check for systemic issues

**Cascading failure prevention:**
- Database persistence prevents data loss
- Cache warming ensures availability
- Circuit breakers prevent retry storms
- Timeouts prevent thread exhaustion

### Database Connection Pool Configuration

**Connection Pool Settings (HikariCP):**

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
For configuration service:
- 4 CPU cores per pod
- Read-heavy (configuration lookups)
- Calculated: (4 × 2) + 1 = 9 connections minimum
- With safety margin: 20 connections per pod
- 80 pods × 20 = 1,600 max connections to database
- Database max_connections: 2,000 (20% headroom)
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
   - PgBouncer pool: 150 connections
   - App pools can share PgBouncer connections efficiently

