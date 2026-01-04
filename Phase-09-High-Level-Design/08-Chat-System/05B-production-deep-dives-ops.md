# Chat System - Production Deep Dives (Operations)

## Overview

This document covers operational aspects: monitoring & metrics, scaling strategies, failure handling, regional deployment, and cost optimization.

---

## 1. Monitoring & Metrics

### Key Metrics

```java
@Component
public class ChatMetrics {
    
    private final MeterRegistry registry;
    
    // Connection metrics
    private final AtomicLong activeConnections = new AtomicLong(0);
    private final Counter connectionAttempts;
    private final Counter connectionFailures;
    
    // Message metrics
    private final Timer messageLatency;
    private final Counter messagesSent;
    private final Counter messagesDelivered;
    private final Counter messagesFailed;
    
    public ChatMetrics(MeterRegistry registry) {
        this.registry = registry;
        
        this.messageLatency = Timer.builder("chat.message.latency")
            .description("Message delivery latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        this.activeConnections = registry.gauge("chat.connections.active", 
            new AtomicLong(0));
    }
    
    public void recordMessageSent(String type) {
        messagesSent.increment();
        registry.counter("chat.messages.sent", "type", type).increment();
    }
    
    public void recordDeliveryLatency(long startTime, boolean success) {
        long duration = System.currentTimeMillis() - startTime;
        messageLatency.record(duration, TimeUnit.MILLISECONDS);
        
        if (success) {
            messagesDelivered.increment();
        } else {
            messagesFailed.increment();
        }
    }
}
```

### Alerting Thresholds

| Metric                  | Warning    | Critical   | Action                        |
| ----------------------- | ---------- | ---------- | ----------------------------- |
| Delivery latency P99    | > 200ms    | > 500ms    | Scale chat servers            |
| Connection failures     | > 1%       | > 5%       | Check load balancer           |
| Message delivery rate   | < 99%      | < 95%      | Check message queue           |
| Pending messages        | > 100K     | > 1M       | Scale delivery workers        |

### Grafana Dashboard Queries

```promql
# Active WebSocket connections
sum(chat_connections_active)

# Message delivery latency P99
histogram_quantile(0.99, sum(rate(chat_message_latency_bucket[5m])) by (le))

# Messages per second
sum(rate(chat_messages_sent_total[1m]))

# Delivery success rate
sum(rate(chat_messages_delivered_total[5m])) / sum(rate(chat_messages_sent_total[5m]))

# Connection failure rate
sum(rate(chat_connection_failures_total[5m])) / sum(rate(chat_connection_attempts_total[5m]))
```

---

## 2. Scaling Strategies

### Connection Scaling

**Single Server Limits:**
- ~100K concurrent WebSocket connections per server
- Limited by file descriptors, memory, CPU

**Scaling Strategy:**

```
150M connections / 100K per server = 1,500 servers minimum
With 50% headroom: 2,250 servers
```

### Architecture for Scale

1. **Stateless Routing Layer:**
   - Load balancer with sticky sessions (by user_id hash)
   - Session registry in Redis for cross-server routing

2. **Horizontal Scaling:**
   - Add more chat servers as users grow
   - Each server handles independent set of users

3. **Connection Optimization:**

```java
// Server-side optimizations
- Epoll/Kqueue for efficient I/O
- Connection pooling for backend services
- Binary protocol (not JSON) for efficiency

// Client-side optimizations
- Reconnect with exponential backoff
- Batch acknowledgments
- Compress large messages
```

### Regional Deployment

```
┌─────────────────────────────────────────────────────────────────┐
│  US-EAST                          │  EU-WEST                     │
│                                   │                              │
│  ┌─────────────┐                  │  ┌─────────────┐            │
│  │ Chat Servers│                  │  │ Chat Servers│            │
│  └─────────────┘                  │  └─────────────┘            │
│         │                         │         │                    │
│  ┌─────────────┐                  │  ┌─────────────┐            │
│  │  Cassandra  │◄────────────────►│  │  Cassandra  │            │
│  │  (Replica)  │   Cross-region   │  │  (Replica)  │            │
│  └─────────────┘   replication    │  └─────────────┘            │
│                                   │                              │
└─────────────────────────────────────────────────────────────────┘
```

**Key Decisions:**

1. **Message Storage:**
   - Write to local region first (low latency)
   - Async replicate to other regions
   - Eventual consistency is acceptable for chat

2. **User Routing:**
   - Route to nearest region by default
   - If chatting with user in different region, route message cross-region

3. **Presence:**
   - Regional presence (local Redis)
   - Cross-region presence sync for contacts in different regions

**Latency Consideration:**
- Same region: ~50ms
- Cross-region: ~150-300ms (acceptable for chat)

### Database Connection Pool Configuration

**PostgreSQL Connection Pool (HikariCP):**

```yaml
spring:
  datasource:
    hikari:
      # Pool sizing
      maximum-pool-size: 25          # Max connections per chat server instance
      minimum-idle: 8                 # Minimum idle connections
      
      # Timeouts
      connection-timeout: 30000       # 30s - Max time to wait for connection from pool
      idle-timeout: 600000            # 10m - Idle connection timeout
      max-lifetime: 1800000           # 30m - Max connection lifetime
      
      # Monitoring
      leak-detection-threshold: 60000 # 60s - Detect connection leaks
      
      # Validation
      connection-test-query: SELECT 1
      validation-timeout: 3000        # 3s - Connection validation timeout
```

**Pool Sizing Calculation:**
```
For chat servers:
- 8 CPU cores per pod
- Spindle count = 1 (single DB instance per shard)
- Calculated: (8 × 2) + 1 = 17 connections minimum
- With safety margin: 25 connections per pod
- 1000 pods × 25 = 25,000 max connections (with connection pooler)
- Use PgBouncer to multiplex connections at database level
```

**Cassandra Connection Pool (Datastax Driver):**

```yaml
datastax-java-driver:
  advanced:
    connection:
      pool:
        local:
          max-requests-per-connection: 32768
          max-connections: 8           # Per node connections
        remote:
          max-requests-per-connection: 32768
          max-connections: 2           # Remote nodes (fewer connections)
    
    # Timeouts
    request:
      timeout: 5s                      # Request timeout
      read-timeout: 5s
      write-timeout: 5s
    
    # Connection monitoring
    heartbeat:
      interval: 30s                    # Keep-alive interval
      timeout: 5s
```

**Pool Exhaustion Handling:**

1. **Monitoring:**
   - Alert when PostgreSQL pool usage > 80%
   - Track Cassandra connection pool metrics
   - Monitor wait time for connections

2. **Circuit Breaker:**
   - If pool exhausted for > 5 seconds, open circuit breaker
   - Fail fast instead of blocking

3. **Connection Pooler (PgBouncer):**
   - Place PgBouncer between chat servers and PostgreSQL
   - PgBouncer pool: 100 connections per PostgreSQL instance
   - Chat server pools can share PgBouncer connections efficiently

### Backup and Recovery

**RPO (Recovery Point Objective):** 0 (no data loss)
- Maximum acceptable data loss
- Messages stored in Cassandra with replication factor 3
- Real-time replication ensures no message loss
- WebSocket connection state can be reconstructed from message history

**RTO (Recovery Time Objective):** < 1 minute
- Maximum acceptable downtime
- Time to restore service via failover to secondary region
- Includes Cassandra failover, WebSocket reconnection, and traffic rerouting

**Backup Strategy:**
- **Cassandra Backups**: Daily full backups, hourly incremental
- **Point-in-time Recovery**: Retention period of 7 days
- **Cross-region Replication**: Real-time replication to secondary region
- **Redis Persistence**: AOF (Append-Only File) with 1-second fsync for presence data

**Restore Steps:**
1. Detect primary region failure (health checks) - 5 seconds
2. Promote Cassandra replica to primary - 10 seconds
3. Update DNS records to point to secondary region - 5 seconds
4. Clients reconnect WebSocket (automatic) - 30 seconds
5. Verify chat service health and resume traffic - 10 seconds

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Cassandra replication lag < 1 minute (synchronous replication)
- [ ] Redis replication lag < 1 minute
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture active connection count
   - Document message delivery rate
   - Verify Cassandra replication lag (< 1 minute)
   - Test message delivery (100 sample messages)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+1 minute):**
   - **T+0-10s:** Detect failure via health checks
   - **T+10-20s:** Promote secondary region to primary
   - **T+20-30s:** Update DNS records (Route53 health checks)
   - **T+30-40s:** Clients reconnect to DR region (automatic)
   - **T+40-50s:** Verify all services healthy in DR region
   - **T+50-60s:** Resume traffic to DR region

4. **Post-Failover Validation (T+1 to T+5 minutes):**
   - Verify RTO < 1 minute: ✅ PASS/FAIL
   - Verify RPO = 0: Check message count matches pre-failover (no loss)
   - Test message delivery: Send 1,000 test messages, verify 100% delivery
   - Test message retrieval: Retrieve 100 random messages, verify 100% success
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify connection recovery: All clients reconnected within 1 minute

5. **Data Integrity Verification:**
   - Compare message count: Pre-failover vs post-failover (should match exactly, RPO=0)
   - Spot check: Verify 100 random messages accessible
   - Check message ordering: Verify messages in correct order
   - Test edge cases: Group chats, media messages, system messages

6. **Failback Procedure (T+5 to T+10 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag < 1 minute
   - Update DNS to route traffic back to primary
   - Monitor for 5 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 1 minute: Time from failure to service resumption
- ✅ RPO = 0: No data loss (verified via message count, exact match)
- ✅ Message delivery works: >99.9% messages delivered correctly
- ✅ No data loss: Message count matches pre-failover exactly
- ✅ Service resumes within RTO target: All metrics return to baseline
- ✅ Connection recovery: All clients reconnected within 1 minute

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: User Sends Message to Friend

**Step-by-step:**

1. **User Action**: User types message "Hello!" and sends via WebSocket
2. **Chat Service** (receives WebSocket message):
   - Validates message format
   - Generates message ID: `msg_xyz789`
   - Stores in Cassandra: `INSERT INTO messages (id, conversation_id, sender_id, content, timestamp)`
   - Publishes to Kafka: `{"event": "message_sent", "message_id": "msg_xyz789", "conversation_id": "conv_123"}`
3. **Response**: WebSocket ACK sent to sender: `{"status": "sent", "message_id": "msg_xyz789"}`
4. **Message Router** (consumes Kafka):
   - Fetches conversation participants: `SELECT user_id FROM conversation_participants WHERE conversation_id = 'conv_123'`
   - For each participant (except sender):
     - Checks if online: `GET user_session:participant_id` → HIT (user online)
     - Routes message via WebSocket to participant's connection
     - Updates read receipt: `SET message_status:msg_xyz789:participant_id "delivered"`
5. **Recipient receives message** via WebSocket: `{"type": "message", "content": "Hello!", "sender": "user123"}`
6. **Recipient reads message**:
   - Client sends read receipt: `{"message_id": "msg_xyz789", "status": "read"}`
   - Updates Cassandra: `UPDATE messages SET read_by = read_by + ['participant_id'] WHERE id = 'msg_xyz789'`
7. **Result**: Message delivered and read within 200ms

**Total latency: ~200ms** (Cassandra write 50ms + Kafka 20ms + WebSocket delivery 130ms)

### Journey 2: Offline User Receives Message

**Step-by-step:**

1. **User Action**: User sends message to friend who is offline
2. **Chat Service**: 
   - Stores message in Cassandra
   - Publishes to Kafka
3. **Message Router**:
   - Checks if recipient online: `GET user_session:recipient_id` → MISS (user offline)
   - Stores in Redis: `LPUSH pending_messages:recipient_id msg_xyz789`
   - Sets TTL: 30 days (message expiration)
4. **Recipient comes online** (30 minutes later):
   - Establishes WebSocket connection
   - Sends: `{"type": "sync", "user_id": "recipient_id"}`
5. **Chat Service**:
   - Fetches pending messages: `LRANGE pending_messages:recipient_id 0 100`
   - Returns: `{"messages": [{"id": "msg_xyz789", "content": "Hello!", ...}, ...]}`
6. **Client**: Displays pending messages in conversation
7. **Result**: Offline user receives all pending messages on reconnect

**Total latency: ~100ms** (Redis lookup + WebSocket delivery)

### Failure & Recovery Walkthrough

**Scenario: Cassandra Node Failure During Message Write**

**RTO (Recovery Time Objective):** < 1 minute (node failover + replica promotion)  
**RPO (Recovery Point Objective):** 0 (replication factor 3, no data loss)

**Timeline:**

```
T+0s:    Cassandra node 5 crashes (handles partition key range 40-50%)
T+0-5s:  Writes to affected partition fail
T+5s:    Cassandra cluster detects node failure
T+10s:   Coordinator routes writes to replicas (nodes 6, 7)
T+15s:   Read repair begins for affected partitions
T+30s:   Node 5 marked as down, replicas handle all traffic
T+60s:   New node provisioned, begins streaming data
T+5min:  New node fully synced, rejoins cluster
```

**What degrades:**
- Writes to partition 40-50% delayed by 10-15 seconds
- Reads may hit replicas (slightly higher latency)
- No data loss (replication factor 3)

**What stays up:**
- Other partitions continue normally
- WebSocket connections unaffected
- Kafka message routing continues
- Most messages delivered successfully

**What recovers automatically:**
- Cassandra cluster promotes replicas
- Coordinator routes around failed node
- New node streams and rejoins
- No manual intervention required

**What requires human intervention:**
- Investigate root cause of node failure
- Replace failed hardware
- Review cluster capacity

**Cascading failure prevention:**
- Replication factor 3 prevents data loss
- Coordinator routing prevents single point of failure
- Circuit breakers prevent retry storms
- Message queuing in Redis for offline users

### Multi-Region Deployment

**Architecture Overview:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PRIMARY REGION: us-east-1                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │  │
│  │  │ Chat        │  │ Chat        │  │ Chat        │           │  │
│  │  │ Server      │  │ Server      │  │ Server      │           │  │
│  │  │ (100 pods)  │  │ (100 pods)  │  │ (100 pods)  │           │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘           │  │
│  │         │                 │                 │                  │  │
│  │  ┌──────┴─────────────────┴─────────────────┴──────┐          │  │
│  │  │  Cassandra (messages) - Multi-DC, sync repl    │          │  │
│  │  │  ──sync repl──> Cassandra (DR region)            │          │  │
│  │  └──────────────────────────────────────────────────┘          │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  Redis Cluster (sessions, presence) - 50 nodes            │ │  │
│  │  │  ──async repl──> Redis Replica (DR)                      │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  S3 Bucket (media storage)                                │ │  │
│  │  │  ──CRR──> S3 Bucket (DR region)                          │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                    ──sync replication──                              │
│                              │                                       │
└──────────────────────────────┼───────────────────────────────────────┘
                               │
┌──────────────────────────────┼───────────────────────────────────────┐
│                    DR REGION: us-west-2                                │
│                              │                                       │
│  ┌───────────────────────────▼─────────────────────────────────────┐ │
│  │  Cassandra Replica (synchronous replication from us-east-1)      │ │
│  │  Redis Replica (async replication + snapshots)                   │ │
│  │  Chat Server (20 pods, minimal for DR readiness)                  │ │
│  │  S3 Bucket (CRR destination, read-only until failover)          │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

**Replication Strategy:**

1. **Cassandra Replication:**
   - **Synchronous:** Primary → DR region (us-west-2) for zero data loss
   - **Replication Method:** Cassandra multi-datacenter replication
   - **Replication Lag:** < 1 minute (synchronous, RPO=0)
   - **Conflict Resolution:** Not applicable (single-writer model, timestamp-based ordering)

2. **Redis Replication:**
   - **Redis Replication:** Async replication to DR region
   - **Snapshots:** Every 5 minutes to S3 for faster recovery
   - **Cache Warm-up:** On failover, restore from latest snapshot

3. **S3 Cross-Region Replication (CRR):**
   - **Replication:** Async replication of media files to DR region
   - **Replication Lag:** Typically < 5 minutes for S3 CRR
   - **Access:** DR region S3 bucket read-only until failover

4. **Traffic Routing:**
   - **Primary:** Route53 health checks route to us-east-1
   - **Failover:** On primary failure, DNS TTL (60s) routes to us-west-2
   - **Geographic Routing:** Optional - route users to nearest region

**Cross-Region Challenges & Solutions:**

| Challenge | Impact | Solution |
|-----------|--------|----------|
| **Synchronous Replication Latency** | Higher write latency (50-100ms) | Acceptable for chat (users expect slight delay), ensures RPO=0 |
| **Connection Migration** | Clients need to reconnect | Automatic reconnection with exponential backoff, < 1 minute RTO |
| **Session State** | Session state may be stale | Acceptable for DR (sessions rebuild on reconnect) |
| **Media Availability** | Media may not be immediately available | Acceptable for DR (RPO < 5 min), critical media can be re-uploaded |
| **Cost** | 2x infrastructure cost | DR region uses smaller instances, only scaled up during failover |
| **Data Consistency** | Strong consistency across regions | Synchronous replication ensures RPO=0, no data loss |

**Failover Scenarios:**

1. **Single Pod Failure:**
   - Impact: Minimal (other pods handle connections)
   - Recovery: Automatic (Kubernetes restarts pod, clients reconnect)
   - RTO: < 40 seconds (automatic)

2. **Entire Region Failure:**
   - Impact: Complete service outage in primary region
   - Recovery: Manual failover to DR region
   - RTO: < 1 minute (manual process)
   - RPO: 0 (synchronous replication, no data loss)

3. **Network Partition:**
   - Impact: Primary region isolated, DR region has latest data (sync repl)
   - Recovery: Manual decision (split-brain prevention)
   - Strategy: Prefer data consistency over availability (CP system)

**Cost Implications:**

- **Primary Region:** $500,000/month (full infrastructure)
- **DR Region:** $150,000/month (reduced capacity, 30% of primary)
- **S3 CRR:** $1,000/month (cross-region replication)
- **Replication Bandwidth:** $5,000/month (cross-region data transfer)
- **Total Multi-Region Cost:** $656,000/month (31% increase)

**Benefits:**
- **Availability:** 99.99% → 99.999% (reduces downtime from 52 min/year to 5 min/year)
- **Disaster Recovery:** Survives region-wide outages
- **Compliance:** Meets regulatory requirements for geographic redundancy
- **Data Durability:** Synchronous replication ensures RPO=0, no data loss

**Trade-offs:**
- **Cost:** 31% increase for DR capability
- **Complexity:** More complex operations (failover procedures, multi-region monitoring)
- **Latency:** Synchronous replication adds 50-100ms latency (acceptable for chat)
- **Consistency:** Strong consistency (RPO=0, no data loss)

---

## 3. Failure Handling

### Scenario 1: Chat Server Crash

**RTO (Recovery Time Objective):** < 40 seconds (client reconnection + sync)  
**RPO (Recovery Point Objective):** 0 (no message loss, messages in Cassandra)

**Problem:** Server with 100K connections crashes.

```java
// Client-side reconnection
public void onDisconnect() {
    int retryCount = 0;
    while (!connected && retryCount < 10) {
        try {
            Thread.sleep(Math.min(1000 * Math.pow(2, retryCount), 30000));
            connect();  // Load balancer routes to healthy server
            sync();     // Get messages since last_sync
        } catch (Exception e) {
            retryCount++;
        }
    }
}

// Server-side: Stateless design
// - Session in Redis (survives server crash)
// - Messages in Cassandra (survives server crash)
// - Pending messages in Kafka (survives server crash)
```

**Recovery Time:**
- Client detects disconnect: ~30 seconds (heartbeat timeout)
- Reconnect to new server: ~1-5 seconds
- Sync missed messages: ~1-2 seconds
- Total: ~35-40 seconds

### Scenario 2: Redis Cluster Failure

**RTO (Recovery Time Objective):** < 30 seconds (automatic failover)  
**RPO (Recovery Point Objective):** 0 (no data loss, sessions may be stale for < 30s)

**Problem:** Session registry unavailable.

**Impact:**
- Can't route messages to correct server
- Can't determine online/offline status

**Solution:**

```java
public void routeMessage(Message message, String recipientId) {
    try {
        String serverId = sessionRegistry.findServer(recipientId)
            .orTimeout(100, TimeUnit.MILLISECONDS)
            .get();
        routeToServer(serverId, message);
    } catch (Exception e) {
        // Fallback: Queue in Kafka, deliver on next connection
        kafkaTemplate.send("pending-messages", recipientId, message);
        log.warn("Redis unavailable, message queued for {}", recipientId);
    }
}
```

**Prevention:**
- Redis Cluster with 3 replicas per shard
- Redis Sentinel for automatic failover
- Local cache for frequently accessed sessions

### Scenario 3: Cassandra Partial Failure

**Problem:** One Cassandra node fails.

**Solution:**
1. Replication factor = 3, so 2 replicas still available
2. Reads/writes continue with CL=QUORUM (2 of 3)
3. Failed node detected by gossip protocol
4. Hints stored on other nodes for failed node
5. When node recovers, hints replayed

**Impact:** None (transparent failover)

### Scenario 4: Message Delivery Timeout

**Problem:** Message sent but no delivery confirmation.

```java
public class MessageDeliveryTracker {
    
    private final Map<String, PendingMessage> pending = new ConcurrentHashMap<>();
    
    public void trackMessage(Message message) {
        pending.put(message.getId(), new PendingMessage(message, Instant.now()));
        
        // Schedule timeout check
        scheduler.schedule(() -> checkDelivery(message.getId()), 
            30, TimeUnit.SECONDS);
    }
    
    public void onDeliveryConfirmed(String messageId) {
        pending.remove(messageId);
    }
    
    private void checkDelivery(String messageId) {
        PendingMessage pm = pending.get(messageId);
        if (pm != null) {
            if (pm.getRetryCount() < 3) {
                // Retry delivery
                pm.incrementRetry();
                deliveryService.deliver(pm.getMessage());
            } else {
                // Give up, message will sync later
                pending.remove(messageId);
                notifySenderFailed(pm.getMessage());
            }
        }
    }
}
```

### Scenario 5: Cassandra Slow Response

**Impact:**
- Message writes delayed
- Message reads for sync delayed
- But: Real-time delivery unaffected (uses Redis)

**Mitigation:**

```java
public CompletableFuture<Void> saveMessage(Message message) {
    // 1. Write to Redis first (fast)
    redis.lpush("recent:" + message.getConversationId(), serialize(message));
    
    // 2. Async write to Cassandra
    return cassandra.saveAsync(message)
        .orTimeout(500, TimeUnit.MILLISECONDS)
        .exceptionally(e -> {
            // Queue for retry
            retryQueue.add(message);
            return null;
        });
}

// Read with fallback
public List<Message> getMessages(String conversationId, int limit) {
    // Try Redis first (recent messages)
    List<Message> recent = redis.lrange("recent:" + conversationId, 0, limit);
    if (recent.size() >= limit) {
        return recent;
    }
    
    // Fall back to Cassandra for older messages
    try {
        return cassandra.getMessages(conversationId, limit)
            .get(200, TimeUnit.MILLISECONDS);
    } catch (TimeoutException e) {
        // Return what we have from Redis
        return recent;
    }
}
```

---

## 4. Graceful Degradation

### Circuit Breaker for Backend Services

```java
@Service
public class ResilientMessageService {
    
    private final CircuitBreaker cassandraBreaker = CircuitBreaker.ofDefaults("cassandra");
    private final CircuitBreaker redisBreaker = CircuitBreaker.ofDefaults("redis");
    
    public void saveMessage(Message message) {
        // Try Cassandra with circuit breaker
        Try<Void> result = Try.ofSupplier(
            CircuitBreaker.decorateSupplier(cassandraBreaker, 
                () -> cassandraRepository.save(message))
        );
        
        if (result.isFailure()) {
            // Fallback: Queue for later persistence
            kafkaTemplate.send("message-persistence-retry", message);
            log.warn("Cassandra unavailable, message queued for retry");
        }
    }
    
    public PresenceStatus getPresence(String userId) {
        // Try Redis with circuit breaker
        return Try.ofSupplier(
            CircuitBreaker.decorateSupplier(redisBreaker,
                () -> presenceService.getPresence(userId))
        ).getOrElse(PresenceStatus.unknown());
    }
}
```

### Degraded Mode Features

| Feature              | Normal Mode                  | Degraded Mode                    |
| -------------------- | ---------------------------- | -------------------------------- |
| Message delivery     | Real-time via WebSocket      | Queue + sync on reconnect        |
| Presence             | Real-time updates            | Show "last seen" only            |
| Typing indicators    | Real-time                    | Disabled                         |
| Read receipts        | Real-time                    | Batch update on sync             |
| Message history      | Instant load                 | Progressive load                 |

## 4. Simulation (End-to-End User Journeys)

### Journey 1: User Sends Message (Happy Path)

**Step-by-step:**

1. **User Action**: User types message in chat app
2. **Client**: Sends message via WebSocket to chat server
3. **Chat Server**: 
   - Validates message
   - Stores in Cassandra (async)
   - Updates Redis session
   - Routes to recipient's server
4. **Recipient Server**: 
   - Checks if user online (Redis session)
   - If online: Push via WebSocket
   - If offline: Queue in Kafka
5. **Response**: Delivery confirmation sent to sender
6. **Recipient**: Receives message in real-time (< 100ms)

### Journey 2: Group Chat Message Broadcast

**Step-by-step:**

1. **User Action**: Sends message to group (100 members)
2. **Chat Server**: 
   - Validates group membership
   - Stores message in Cassandra
   - Looks up all 100 members
3. **Routing**: 
   - 80 members online → Push via WebSocket
   - 20 members offline → Queue in Kafka
4. **Delivery**: 
   - Online members receive in < 100ms
   - Offline members receive on next login
5. **Response**: Delivery receipts from online members

### High-Load / Contention Scenario: Viral Message Storm

```
Scenario: Celebrity sends message to group with 1M members
Time: Breaking news event (peak traffic)

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Celebrity sends message to 1M member group          │
│ - Message: "Breaking news announcement!"                   │
│ - Group size: 1,000,000 members                           │
│ - Expected: ~100 messages/second normal traffic            │
│ - 10,000x traffic spike                                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+0-5s: Message Distribution Storm                        │
│ - Message stored in Cassandra (single write)              │
│ - Lookup 1M members from database                        │
│ - Route to 1M recipients                                  │
│ - Online members: ~100K (10% online rate)                │
│ - Offline members: ~900K (queue in Kafka)                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Load Handling:                                             │
│                                                              │
│ 1. Message Storage:                                       │
│    - Single message stored in Cassandra                   │
│    - Replication factor 3 (durable)                        │
│    - Storage: ~1KB per message                            │
│                                                              │
│ 2. Online Delivery (100K members):                        │
│    - Batch routing: 1000 members per batch                │
│    - Each batch: 10ms processing                         │
│    - Total: 100 batches × 10ms = 1 second                │
│    - All online members receive in < 2 seconds           │
│                                                              │
│ 3. Offline Queueing (900K members):                      │
│    - Kafka: 900K messages produced                       │
│    - Partitioned by user_id (even distribution)           │
│    - Kafka handles 1M messages/second (safe)              │
│    - Queue processing: 10K messages/second               │
│    - All queued within 90 seconds                        │
│                                                              │
│ 4. Database Load:                                         │
│    - Single message write (no issue)                     │
│    - 1M member lookups: Cached (Redis)                    │
│    - Cache hit rate: 95% (group membership cached)        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - All 100K online members receive in < 2 seconds         │
│ - All 900K offline members queued in < 90 seconds        │
│ - P99 delivery latency: 1.5 seconds (online)            │
│ - No service degradation                                   │
│ - System handles 1M member group gracefully              │
└─────────────────────────────────────────────────────────────┘
```

### Edge Case Scenario: Message Ordering in Group Chat

```
Scenario: Multiple users send messages simultaneously in group chat
Edge Case: Messages arrive out of order due to network delays

┌─────────────────────────────────────────────────────────────┐
│ T+0ms: User A sends message "Message A"                   │
│ - Server timestamp: T0                                    │
│ - Message ID: msg_a_001                                  │
│ - Sent to group (100 members)                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+50ms: User B sends message "Message B" (concurrent)    │
│ - Server timestamp: T50                                   │
│ - Message ID: msg_b_002                                  │
│ - Sent to group (same 100 members)                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Network Delay Issue:                                       │
│                                                              │
│ User C receives:                                           │
│ - Message B at T+100ms (arrives first)                   │
│ - Message A at T+150ms (arrives second, but sent first)  │
│                                                              │
│ Problem: Messages displayed out of order                  │
│ - User sees: "Message B" then "Message A"                │
│ - But "Message A" was sent first                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Solution: Vector Clocks for Message Ordering              │
│                                                              │
│ 1. Vector Clock Implementation:                          │
│    - Each message has vector clock: {server_id: timestamp}│
│    - Message A: {server_1: T0}                           │
│    - Message B: {server_1: T50}                          │
│                                                              │
│ 2. Client-Side Ordering:                                 │
│    - Client maintains received vector clock              │
│    - Compare vector clocks to determine order            │
│    - If A < B (vector clock comparison), A comes first  │
│                                                              │
│ 3. Server-Side Ordering (Alternative):                   │
│    - Use Lamport timestamps (simpler)                    │
│    - Server assigns monotonic timestamp per group        │
│    - Messages ordered by timestamp on server             │
│    - Clients receive in server-assigned order            │
│                                                              │
│ 4. Display Logic:                                         │
│    - Messages sorted by timestamp before display         │
│    - UI shows messages in chronological order            │
│    - Out-of-order messages re-ordered on client          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - Messages displayed in correct order (A before B)       │
│ - Network delays don't affect ordering                   │
│ - User experience: Consistent message order              │
│ - No message loss or duplication                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Cost Optimization

**Current Monthly Cost:** $500,000
**Target Monthly Cost:** $350,000 (30% reduction)

**Top 3 Cost Drivers:**
1. **Chat Servers (EC2):** $200,000/month (40%) - Largest single component
2. **Cassandra Storage:** $150,000/month (30%) - Large message storage
3. **Network (Data Transfer):** $50,000/month (10%) - High data transfer volume

**Optimization Strategies (Ranked by Impact):**

1. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand instances for chat servers
   - **Optimization:** 1-year Reserved Instances for 80% of compute
   - **Savings:** $80,000/month (40% of $200K)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

2. **Message Storage Optimization (40% savings on storage):**
   - **Current:** All messages in standard storage
   - **Optimization:** 
     - Compress messages before storage (50% reduction)
     - TTL for old messages (move to S3 Glacier after 1 year)
     - Archive old conversations to cold storage
   - **Savings:** $60,000/month (40% of $150K storage)
   - **Trade-off:** Slight CPU overhead for compression (negligible)

3. **Connection Efficiency (20% savings on compute):**
   - **Current:** JSON WebSocket frames
   - **Optimization:** 
     - Binary protocol instead of JSON (30% smaller payloads)
     - Connection pooling (reduce server count by 20%)
   - **Savings:** $40,000/month (20% of $200K compute)
   - **Trade-off:** Slight complexity increase (acceptable)

4. **Redis Optimization (30% savings):**
   - **Current:** All sessions cached in Redis
   - **Optimization:** 
     - TTL for sessions (evict inactive sessions)
     - Use smaller instance types where possible
   - **Savings:** $15,000/month (30% of $50K Redis)
   - **Trade-off:** Slight latency for cold sessions (acceptable)

5. **Network Optimization (30% savings):**
   - **Current:** High cross-region data transfer costs
   - **Optimization:** 
     - Regional deployment (route users to nearest region)
     - CDN for media files
   - **Savings:** $15,000/month (30% of $50K network)
   - **Trade-off:** Slight complexity increase (acceptable)

6. **Push Notification Optimization:**
   - **Current:** Individual push notifications
   - **Optimization:** 
     - Batch notifications (reduce API calls by 50%)
     - Smart grouping (reduce redundant notifications)
   - **Savings:** $5,000/month (20% of $25K push notifications)
   - **Trade-off:** Slight delay for batched notifications (acceptable)

**Total Potential Savings:** $215,000/month (43% reduction)
**Optimized Monthly Cost:** $285,000/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Message Delivery | $0.00002 | $0.000011 | 45% |
| Media Storage (per GB/month) | $0.023 | $0.014 | 39% |
| Connection (per hour) | $0.0001 | $0.00006 | 40% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Reserved Instances, Message Storage Optimization → $140K savings
2. **Phase 2 (Month 2):** Connection Efficiency, Redis Optimization → $55K savings
3. **Phase 3 (Month 3):** Network Optimization, Push Notification Optimization → $20K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor message delivery latency (ensure no degradation)
- Monitor storage costs (target < $90K/month)
- Review and adjust quarterly

### Infrastructure Cost Breakdown

| Component        | Cost Driver                | Optimization Strategy                |
| ---------------- | -------------------------- | ------------------------------------ |
| Chat servers     | CPU, memory per connection | Binary protocol, connection pooling  |
| Cassandra        | Storage, IOPS              | Compression, TTL for old messages    |
| Redis            | Memory                     | TTL for sessions, eviction policies  |
| Network          | Cross-AZ/region traffic    | Regional deployment, CDN for media   |
| Push notifications| Per-notification cost     | Batch notifications, smart grouping  |

### Message Storage Optimization

```java
// Compress messages before storage
public byte[] compressMessage(Message message) {
    byte[] json = objectMapper.writeValueAsBytes(message);
    return Snappy.compress(json);  // ~50% compression for text
}

// TTL for old messages
CREATE TABLE messages (
    ...
) WITH default_time_to_live = 31536000;  // 1 year TTL

// Archive old conversations to cold storage
@Scheduled(cron = "0 0 3 * * *")  // 3 AM daily
public void archiveOldMessages() {
    List<Message> oldMessages = cassandra.getMessagesOlderThan(
        Instant.now().minus(365, ChronoUnit.DAYS)
    );
    
    // Move to S3 Glacier
    s3Client.putObject(archiveBucket, archiveKey, serialize(oldMessages));
    
    // Delete from Cassandra
    cassandra.deleteMessages(oldMessages);
}
```

### Connection Efficiency

```java
// Binary WebSocket frames instead of JSON
public class BinaryMessageCodec {
    
    public byte[] encode(ChatMessage message) {
        ByteBuffer buffer = ByteBuffer.allocate(estimateSize(message));
        buffer.put(message.getType().getCode());
        buffer.putLong(message.getTimestamp());
        writeString(buffer, message.getConversationId());
        writeString(buffer, message.getContent());
        return buffer.array();
    }
    
    // ~40% smaller than JSON for typical messages
}
```

---

## 6. Capacity Planning

### Calculation Example

**Assumptions:**
- 100M daily active users
- 50% concurrent at peak = 50M connections
- Average 50 messages/user/day = 5B messages/day
- Average message size: 200 bytes

**Server Capacity:**
```
Chat servers: 50M / 100K = 500 servers (+ 50% headroom = 750)
```

**Storage Capacity:**
```
Daily messages: 5B × 200 bytes = 1TB/day
With replication (RF=3): 3TB/day
Monthly: 90TB
Yearly: 1PB (before compression)
With compression: ~500TB/year
```

**Redis Capacity:**
```
Sessions: 50M × 500 bytes = 25GB
Presence: 100M × 50 bytes = 5GB
Pending queues: 10M × 1KB = 10GB
Total: ~50GB (with headroom: 100GB cluster)
```

---

## 7. Health Checks

### Service Health Endpoints

```java
@RestController
@RequestMapping("/health")
public class HealthController {
    
    @GetMapping("/live")
    public ResponseEntity<String> liveness() {
        return ResponseEntity.ok("OK");
    }
    
    @GetMapping("/ready")
    public ResponseEntity<HealthStatus> readiness() {
        HealthStatus status = HealthStatus.builder()
            .cassandra(checkCassandra())
            .redis(checkRedis())
            .kafka(checkKafka())
            .build();
        
        if (status.isHealthy()) {
            return ResponseEntity.ok(status);
        }
        return ResponseEntity.status(503).body(status);
    }
    
    private ComponentHealth checkCassandra() {
        try {
            cassandra.execute("SELECT now() FROM system.local");
            return ComponentHealth.healthy();
        } catch (Exception e) {
            return ComponentHealth.unhealthy(e.getMessage());
        }
    }
    
    private ComponentHealth checkRedis() {
        try {
            redis.ping();
            return ComponentHealth.healthy();
        } catch (Exception e) {
            return ComponentHealth.unhealthy(e.getMessage());
        }
    }
}
```

### WebSocket Health Monitoring

```java
@Component
public class WebSocketHealthMonitor {
    
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    
    @PostConstruct
    public void startMonitoring() {
        scheduler.scheduleAtFixedRate(this::checkConnections, 0, 30, TimeUnit.SECONDS);
    }
    
    private void checkConnections() {
        int active = sessionRegistry.getLocalSessionCount();
        int stale = sessionRegistry.getStaleSessionCount(Duration.ofMinutes(5));
        
        if (stale > active * 0.1) {
            log.warn("High stale session count: {} of {}", stale, active);
            // Clean up stale sessions
            sessionRegistry.cleanupStaleSessions();
        }
        
        metrics.gauge("chat.sessions.active", active);
        metrics.gauge("chat.sessions.stale", stale);
    }
}
```

---

## Summary

| Aspect              | Strategy                                              |
| ------------------- | ----------------------------------------------------- |
| Monitoring          | Prometheus + Grafana, custom chat metrics             |
| Scaling             | Horizontal (100K conn/server), regional deployment    |
| Failure handling    | Circuit breakers, graceful degradation, retry queues  |
| Cost optimization   | Binary protocol, compression, TTL, cold storage       |
| Capacity planning   | 750 servers for 50M concurrent, 500TB/year storage    |
| Health checks       | Liveness/readiness probes, connection monitoring      |

