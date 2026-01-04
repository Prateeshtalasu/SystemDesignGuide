# Distributed Configuration Management - Production Deep Dives: Core

## STEP 6: ASYNC, MESSAGING & EVENT FLOW

### A) CONCEPT: What is Kafka?

Kafka is a distributed event streaming platform for high-throughput, durable event distribution.

**Why Async for Configuration Updates:**
- Updates must propagate to 10K services
- Synchronous updates would be slow and unreliable
- Async allows decoupling and retry

### B) OUR USAGE

**Topic Design:**
```
Topic: config.updates
  Partitions: 10
  Replication: 3
  Retention: 7 days
```

**Partition Key: `key` (configuration key)**
- Ensures same key updates are ordered
- Enables per-key processing

**Consumer Groups:**
- Update Propagation Service: Consumes updates, pushes to WebSocket clients
- Audit Service: Logs all changes
- Notification Service: Sends alerts on critical changes

### C) STEP-BY-STEP SIMULATION

**Configuration Update Flow:**

```
1. Admin updates config
   PUT /v1/config/services.payment.database_url
   ↓
2. Configuration Service writes to PostgreSQL
   - New version created
   - Current version updated
   ↓
3. Publish to Kafka
   Topic: config.updates
   Partition: hash(key) % 10
   Message: {key, value, version, timestamp}
   ↓
4. Update Propagation Service consumes
   - Reads from Kafka
   - Finds subscribed WebSocket clients
   - Pushes update to clients
   ↓
5. Services receive update
   - WebSocket message received
   - Service reloads configuration
   - Update complete (< 5 seconds)
```

**Failure Handling:**
- Kafka down: Updates queued, retried when Kafka recovers
- WebSocket client disconnected: Update queued, delivered on reconnect
- Service crash: Service polls on restart, gets latest config

---

## STEP 7: CACHING (REDIS)

### A) CONCEPT

Redis provides fast in-memory caching for configuration reads.

### B) OUR USAGE

**Cache Keys:**
```
config:{key}:{environment} → {value, version}
config:version:{key}:{environment} → {version}
```

**TTL:** 1 hour (configs change infrequently)

**Eviction:** `allkeys-lru`

**Invalidation:** On update, delete cache key, publish to Redis pub/sub

### C) STEP-BY-STEP SIMULATION

**Cache HIT Path:**
```
1. Service requests config
   GET /v1/config/services.payment.database_url
   ↓
2. Check Redis
   Key: config:services.payment.database_url:production
   Result: HIT
   ↓
3. Return from cache
   Latency: < 5ms
```

**Cache MISS Path:**
```
1. Service requests config
   ↓
2. Check Redis
   Result: MISS
   ↓
3. Query PostgreSQL
   SELECT * FROM configurations WHERE key = '...' AND environment = 'production'
   ↓
4. Cache result
   SET config:services.payment.database_url:production {value, version}
   EXPIRE 3600
   ↓
5. Return result
   Latency: < 50ms
```

**What if Redis is Down?**
- Fallback: Query PostgreSQL directly
- Latency: < 100ms (acceptable)
- No data loss (PostgreSQL is source of truth)

---

## STEP 8: SEARCH

**Not Required:**
- Configurations are accessed by key (not searched)
- Use PostgreSQL indexes for key lookups
- No full-text search needed

**If Needed Later:**
- Add Elasticsearch for configuration search
- Index configuration keys and values
- Support fuzzy search

---

## Summary

**Async/Messaging:**
- Kafka for update propagation
- WebSocket for real-time delivery
- Pub/sub for cache invalidation

**Caching:**
- Redis for fast reads
- 1-hour TTL
- Invalidate on update

**Search:**
- Not required (key-based access)

