# Distributed Configuration Management - Interview Grilling

## STEP 14: TRADEOFFS & INTERVIEW QUESTIONS

### Why PostgreSQL over etcd/Consul?

**PostgreSQL:**
- Pros: ACID, SQL queries, versioning, mature
- Cons: Higher latency than etcd
- **We chose PostgreSQL** for ACID guarantees and versioning

**etcd:**
- Pros: Fast, distributed, used by Kubernetes
- Cons: Limited querying, no SQL
- **Why not:** Need SQL queries and versioning

**Consul:**
- Pros: Service discovery, health checks
- Cons: Less focused on configuration
- **Why not:** Need dedicated config management

### Why Strong Consistency?

**Answer:** Configuration changes must be immediately visible. Stale configuration can cause service failures. Better to fail than serve wrong configuration.

### What Breaks First Under Load?

**Answer:** Redis cache. If cache is overwhelmed, all reads go to PostgreSQL, causing database overload.

**Mitigation:** Scale Redis, increase cache TTL, optimize cache keys

### System Evolution

**MVP:**
- Simple key-value store
- File-based storage
- No versioning

**Production:**
- PostgreSQL for storage
- Redis for caching
- Versioning and rollback
- Real-time updates

**Scale:**
- Sharding for large scale
- Multi-region replication
- Advanced features (feature flags, secrets)

### Level-Specific Answers

**L4:** Understand basic architecture, caching, versioning
**L5:** Understand production concerns, failure scenarios, scaling
**L6:** Understand multiple architectures, migrations, org-level trade-offs

## STEP 15: FAILURE SCENARIOS

**PostgreSQL Failure:**
- Automatic failover to replica
- No data loss
- Writes resume in < 30 seconds

**Redis Failure:**
- Fallback to PostgreSQL
- Higher latency (acceptable)
- Cache repopulated on recovery

**Kafka Failure:**
- Updates queued
- Services poll for updates
- Updates delivered when Kafka recovers

**WebSocket Service Failure:**
- Services fall back to polling
- Updates delivered on next poll
- No data loss

**Multi-Region Failure:**
- DNS failover to secondary region
- Configs replicated across regions
- RTO: 1 hour, RPO: 5 minutes

