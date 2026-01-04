# Key-Value Store - Production Deep Dives: Operations

## STEP 9: SCALING & RELIABILITY

**Horizontal Scaling:** Add nodes, rebalance slots
**Load Balancing:** Client-side sharding (consistent hashing)
**Replication:** Master-replica (3x replication)
**Failover:** Automatic (Sentinel, < 30 seconds)
**Eviction:** LRU/LFU when memory full

## STEP 10: MONITORING & OBSERVABILITY

**Key Metrics:**
- Latency: p99 < 1ms
- Throughput: 1M ops/second
- Memory usage: < 80%
- Hit rate: > 90% (for caching)
- Replication lag: < 1 second

**Alerting:**
- Critical: Node down, high memory, high latency
- Warning: Replication lag, eviction rate

**Logging:**
- Command logs (optional)
- Error logs
- Slow query logs (> 10ms)

## STEP 11: SECURITY

**Authentication:** AUTH password
**Authorization:** ACL (Access Control Lists)
**Encryption:** TLS in transit
**Network:** Firewall rules, VPC isolation

## STEP 12: SIMULATION

**User Journey: Cache Database Query**

1. Application queries database (slow, 100ms)
2. Application checks key-value store (GET query:123)
3. Cache MISS: Query database, store result (SET query:123 "result" EX 3600)
4. Next request: Cache HIT (< 1ms)
5. After 1 hour: Key expires, cache MISS again

**Failure Scenario: Master Node Fails**

- Sentinel detects failure
- Promotes replica to master (< 30 seconds)
- Clients reconnect to new master
- No data loss (replication)
- Slight availability impact during failover

## STEP 13: COST ANALYSIS

**Storage:** ~$175/month (memory + disk)
**Compute:** ~$10K/month (20 nodes)
**Total:** ~$10,200/month

### Cost Optimization Strategies

**Current Monthly Cost:** $10,200
**Target Monthly Cost:** $7,140 (30% reduction)

**Top 3 Cost Drivers:**
1. **Compute (EC2):** $10,000/month (98%) - Largest single component
2. **Storage (Memory + Disk):** $175/month (2%) - Redis storage
3. **Network (Data Transfer):** $25/month (<1%) - Data transfer costs

**Optimization Strategies (Ranked by Impact):**

1. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand instances
   - **Optimization:** 1-year Reserved Instances for 80% of compute
   - **Savings:** $4,000/month (40% of $10K compute)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

2. **Spot Instances for Replicas (60% savings):**
   - **Current:** On-demand instances for all nodes
   - **Optimization:** 
     - Use spot instances for replicas (60% savings)
     - Keep primary nodes on-demand for reliability
   - **Savings:** $1,800/month (60% of $3K replica compute)
   - **Trade-off:** Possible interruptions (acceptable, replicas can be replaced)

3. **Memory Optimization (20% savings):**
   - **Current:** Over-provisioned memory
   - **Optimization:** 
     - Optimize memory usage
     - Use smaller instances for non-critical data
   - **Savings:** $2,000/month (20% of $10K compute)
   - **Trade-off:** Need careful monitoring to avoid memory issues

4. **Right-sizing:**
   - **Current:** Over-provisioned instances
   - **Optimization:** 
     - Monitor actual usage for 1 month
     - Reduce instance sizes where possible
   - **Savings:** $1,000/month (10% of $10K compute)
   - **Trade-off:** Need careful monitoring to avoid performance issues

**Total Potential Savings:** $8,800/month (86% reduction)
**Optimized Monthly Cost:** $1,400/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| GET Operation | $0.00000001 | $0.0000000014 | 86% |
| SET Operation | $0.00000001 | $0.0000000014 | 86% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Reserved Instances, Spot Instances → $5.8K savings
2. **Phase 2 (Month 2):** Memory Optimization, Right-sizing → $3K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor key-value operation latency (ensure no degradation)
- Monitor memory usage (target < 80%)
- Review and adjust quarterly

**Optimization:**
- Use smaller instances for non-critical data
- Optimize memory usage
- Use spot instances for replicas

---

## Backup and Recovery

**RPO (Recovery Point Objective):** 0 (no data loss)
- Maximum acceptable data loss
- Data stored with replication factor 3
- All replicas must acknowledge before commit
- Cross-region replication ensures no data loss

**RTO (Recovery Time Objective):** < 1 minute
- Maximum acceptable downtime
- Time to restore service via automatic failover
- Includes replica promotion and client reconnection

**Backup Strategy:**
- **Data Replication**: 3x replication across nodes
- **Cross-region Replication**: Real-time replication to DR region
- **Snapshot Backups**: Daily snapshots to S3 (for disaster recovery)
- **Persistence**: AOF (Append-Only File) with 1-second fsync

**Restore Steps:**
1. Detect node failure (health checks) - 5 seconds
2. Automatic failover to replica - 10 seconds
3. Replica promoted to master - 15 seconds
4. Clients reconnect to new master - 20 seconds
5. Verify key-value store health - 10 seconds

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Redis replication lag = 0 (synchronous replication)
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture key-value operation success rate
   - Document active key count
   - Verify Redis replication lag (= 0, synchronous)
   - Test key-value operations (100 sample operations)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+1 minute):**
   - **T+0-5s:** Detect node failure via health checks
   - **T+5-10s:** Automatic failover to replica
   - **T+10-15s:** Replica promoted to master
   - **T+15-20s:** Clients reconnect to new master
   - **T+20-30s:** Verify key-value store health
   - **T+30-60s:** Resume traffic to DR region

4. **Post-Failover Validation (T+1 to T+5 minutes):**
   - Verify RTO < 1 minute: ✅ PASS/FAIL
   - Verify RPO = 0: Check key count matches pre-failover (exact match)
   - Test key-value operations: Perform 1,000 test operations, verify 100% success
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify key integrity: Check all keys accessible

5. **Data Integrity Verification:**
   - Compare key count: Pre-failover vs post-failover (should match exactly, RPO=0)
   - Spot check: Verify 100 random keys accessible
   - Test edge cases: High-throughput operations, large values, TTL expiration

6. **Failback Procedure (T+5 to T+6 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag = 0
   - Update DNS to route traffic back to primary
   - Monitor for 1 minute before declaring success

**Validation Criteria:**
- ✅ RTO < 1 minute: Time from failure to service resumption
- ✅ RPO = 0: No data loss (verified via key count, exact match)
- ✅ Key-value operations work: 100% operations successful
- ✅ No data loss: Key count matches pre-failover exactly
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

### Journey 1: Client Sets and Gets Value

**Step-by-step:**

1. **Client sets value**: Client sends `SET user:123 "John Doe"` via RESP protocol
2. **Key-Value Store**: 
   - Parses command: `SET user:123 "John Doe"`
   - Calculates hash: `hash("user:123") % 10 = 5`
   - Routes to node 5
   - Stores in memory: `SET user:123 "John Doe"`
   - Appends to AOF: `*3\r\n$3\r\nSET\r\n$8\r\nuser:123\r\n$8\r\nJohn Doe\r\n`
   - Returns: `+OK\r\n`
3. **Response**: `+OK` (success)
4. **Client gets value** (immediately after):
   - Client sends: `GET user:123`
   - Key-Value Store routes to node 5
   - Retrieves from memory: `GET user:123` → Returns "John Doe"
   - Returns: `$8\r\nJohn Doe\r\n`
5. **Response**: `$8\r\nJohn Doe\r\n` (value)
6. **Result**: Value set and retrieved successfully, < 1ms latency

**Total latency: < 1ms** (in-memory operation)

### Journey 2: Replication and Failover

**Step-by-step:**

1. **Primary node receives write**: `SET user:123 "John Doe"` on node 5 (primary)
2. **Replication**:
   - Primary writes to AOF
   - Sends to replicas: `REPLICA node_6: SET user:123 "John Doe"`
   - Replica 6 acknowledges: `+OK`
   - Replica 7 acknowledges: `+OK`
   - Primary confirms: 2/2 replicas acknowledged
3. **Primary fails** (node 5 crashes):
   - Health check fails
   - Coordinator detects failure
   - Promotes replica 6 to primary
4. **Client retries**:
   - Client sends: `GET user:123`
   - Routes to new primary (node 6)
   - Returns: `$8\r\nJohn Doe\r\n`
5. **Result**: Replication ensures data availability, failover transparent to client

**Total latency: Replication ~10ms, Failover ~2s, Query < 1ms**

### Failure & Recovery Walkthrough

**Scenario: Primary Node Failure**

**RTO (Recovery Time Objective):** < 2 minutes (replica promotion + client reconnection)  
**RPO (Recovery Point Objective):** < 10ms (sync replication, no data loss)

**Timeline:**

```
T+0s:    Primary node 5 crashes
T+0-5s:  Writes to node 5 fail
T+5s:    Coordinator detects failure
T+10s:   Replica 6 promoted to primary
T+15s:   Cluster topology updated
T+20s:   Clients reconnect to new primary
T+30s:   All operations resume
T+60s:   New replica provisioned, begins sync
T+2min:  New replica fully synced, rejoins cluster
```

**What degrades:**
- Writes to node 5 fail for 10-15 seconds
- Replica promotion takes 10 seconds
- No data loss (sync replication)

**What stays up:**
- Reads from replicas (unaffected)
- Writes to other nodes (unaffected)
- All other operations

**What recovers automatically:**
- Coordinator promotes replica to primary
- Clients reconnect automatically
- New replica syncs and rejoins
- No manual intervention required

**What requires human intervention:**
- Investigate root cause of node failure
- Replace failed hardware
- Review node capacity

**Cascading failure prevention:**
- Sync replication prevents data loss
- Replica promotion ensures availability
- Circuit breakers prevent retry storms
- Timeouts prevent thread exhaustion

### Database Connection Pool Configuration

**Redis Connection Pool (Jedis):**

```yaml
redis:
  jedis:
    pool:
      max-active: 100            # Max connections per instance
      max-idle: 50                # Max idle connections
      min-idle: 20                # Min idle connections
      
    # Timeouts
    connection-timeout: 2000      # 2s - Max time to get connection
    socket-timeout: 3000          # 3s - Max time for operation
```

**Pool Sizing Calculation:**
```
For key-value store clients:
- 8 CPU cores per application instance
- Very high read/write rate (key-value operations)
- Calculated: 100 connections per application instance
- 100 application instances × 100 = 10,000 max connections
- Key-value store cluster: 20 nodes, 500 connections per node capacity
```

**Pool Exhaustion Mitigation:**

1. **Monitoring:**
   - Alert when pool usage > 80%
   - Track wait time for connections
   - Monitor connection leak detection

2. **Circuit Breaker:**
   - If pool exhausted for > 5 seconds, open circuit breaker
   - Fail fast instead of blocking

3. **Connection Pooling:**
   - Use connection pooling libraries (Jedis pool)
   - Monitor pool metrics and adjust based on load

