# URL Shortener - Interview Grilling Q&A

## Trade-off Questions

### Q1: Why PostgreSQL over NoSQL (DynamoDB, MongoDB)?

**Answer:**

```
PostgreSQL is chosen because:

1. ACID Transactions: URL creation needs atomicity. If we insert a URL but fail
   to cache it, we need to rollback. PostgreSQL handles this natively.

2. Strong Consistency: When a user creates a URL, they must see it immediately.
   PostgreSQL's synchronous replication guarantees this.

3. Complex Queries: Analytics queries (GROUP BY, aggregations) are easier in SQL.
   NoSQL would require pre-aggregation or external processing.

4. Mature Ecosystem: Battle-tested replication, backup, monitoring tools.

When would I choose DynamoDB?
- If we needed automatic scaling without ops overhead
- If access patterns were purely key-value (no analytics)
- If we were already heavily invested in AWS
```

### Q2: Why Redis over Memcached?

**Answer:**

```
Redis advantages for this use case:

1. Data Structures: We use sorted sets for rate limiting (sliding window).
   Memcached only supports simple key-value.

2. Persistence: Redis can persist data. If Redis restarts, we don't lose
   the entire cache (RDB snapshots).

3. Cluster Mode: Redis Cluster provides automatic sharding and failover.
   Memcached requires client-side sharding.

4. Pub/Sub: We can use Redis pub/sub for cache invalidation across
   multiple app servers.

When would I choose Memcached?
- Pure caching with no persistence needs
- Simpler operational model
- Lower memory overhead per key
```

### Q3: Why 301 redirect instead of 302?

**Answer:**

```
301 (Permanent Redirect):
- Browsers cache the redirect
- Reduces load on our servers
- Better for SEO (link juice passes through)

302 (Temporary Redirect):
- Browser always hits our server
- We can track every click
- We can change destination without cache issues

Our choice: 301 by default, 302 optional

Reasoning:
- Most URLs don't change destination
- CDN caching with 301 reduces latency
- For analytics, we track on first request, then let browsers cache
- Users who need tracking can choose 302
```

### Q4: What if Redis goes down?

**Answer:**

```
Failure scenario: Redis cluster becomes unavailable

Impact:
- Cache misses for all requests
- All 12,000 QPS hit PostgreSQL directly
- PostgreSQL can handle ~5,000 QPS per replica
- With 2 replicas: 10,000 QPS capacity
- System degrades but doesn't fail completely

Mitigation strategies:

1. Redis Cluster with 3+ nodes:
   - Automatic failover if one node dies
   - No single point of failure

2. Circuit Breaker:
   - If Redis errors > threshold, bypass cache
   - Go directly to database
   - Prevents cascading failures

3. Local Cache (Caffeine):
   - Keep hot URLs in JVM memory
   - Reduces Redis dependency for top 1000 URLs

4. Graceful Degradation:
   - Serve redirects without analytics
   - Queue click events locally, replay later
```

---

## Scaling Questions

### Q5: How would you handle 100x traffic increase?

**Answer:**

```
Current: 12,000 QPS peak
Target: 1,200,000 QPS peak

Changes needed:

1. CDN Layer:
   - Increase edge cache TTL
   - Add more edge locations
   - Enable aggressive caching
   - Target: 80% requests served from CDN

2. Application Layer:
   - Scale from 10 to 100+ pods
   - Use Kubernetes HPA for auto-scaling
   - Optimize JVM for lower latency

3. Cache Layer:
   - Scale Redis cluster from 6 to 30+ nodes
   - Increase memory per node
   - Use Redis read replicas

4. Database Layer:
   - Add more read replicas (10+)
   - Consider sharding by short_code hash
   - Use connection pooling (PgBouncer)

5. Analytics:
   - Scale Kafka partitions to 100+
   - Use ClickHouse for real-time analytics
   - Aggressive aggregation (minute-level instead of raw)

Cost implication:
- Current: ~$10,000/month
- At 100x: ~$200,000/month (not linear due to efficiencies)
```

### Q6: What's the first thing that breaks under load?

**Answer:**

```
Bottleneck analysis (in order of failure):

1. Database Connections (First to break)
   - Default: 100 connections per PostgreSQL
   - 10 app servers × 50 connections = 500 connections needed
   - Solution: PgBouncer connection pooling

2. Cache Memory
   - Hot URLs exceed cache capacity
   - Cache eviction increases, hit rate drops
   - Database gets overwhelmed
   - Solution: Increase cache size, add nodes

3. Kafka Consumer Lag
   - Analytics events pile up
   - Consumer can't keep up with producers
   - Solution: Add consumers, increase partitions

4. Network Bandwidth
   - Usually not a problem for URL shortener
   - Responses are small (< 1KB)
   - Would need 10+ Gbps to saturate

5. CPU on App Servers
   - URL shortening is not CPU-intensive
   - Mostly I/O bound
   - Would need extreme load to saturate
```

### Q7: How do you handle a celebrity tweet? (Thundering Herd)

**Answer:**

```
Scenario: Celebrity with 50M followers tweets a short URL
Expected: 1M+ clicks in first minute

Problems:
1. CDN cache is cold for this URL
2. All requests hit origin simultaneously
3. Database overwhelmed by single URL lookup

Solutions:

1. Request Coalescing:
   - Multiple requests for same URL wait for one DB query
   - Use singleflight pattern (Go) or similar in Java

   @Cacheable(sync = true)  // Spring's synchronized cache
   public String getOriginalUrl(String shortCode) {
       return urlRepository.findByShortCode(shortCode);
   }

2. CDN Cache Warming:
   - When URL is created, pre-warm CDN edges
   - Push to edge locations proactively

3. Local Cache:
   - Hot URLs cached in JVM (Caffeine)
   - Reduces Redis round-trip
   - Even 1-second TTL helps with thundering herd

4. Rate Limiting per URL:
   - Limit redirects per URL per second
   - Spread load over time
   - Return 429 with Retry-After header

5. Graceful Degradation:
   - If URL lookup fails, serve static page
   - "High traffic, try again in a moment"
```

---

## Failure Scenarios

### Q8: What happens if the primary database fails?

**Answer:**

```
Failure: PostgreSQL primary becomes unavailable

Timeline:

0s:     Primary fails
0-5s:   Health checks detect failure
5-10s:  Streaming replication to sync replica stops
10-30s: Automatic failover promotes sync replica to primary
30-60s: Application reconnects to new primary
60s+:   System fully operational

Impact during failover:
- Reads: Continue working (replicas available)
- Writes: Fail for 30-60 seconds
- URL creation returns 503 Service Unavailable

Data loss:
- Zero with synchronous replication (our setup)
- Up to a few seconds with async replication

Recovery steps:
1. Automatic: Patroni/pg_auto_failover handles promotion
2. DNS/endpoint updated automatically
3. Old primary recovered as new replica
4. Post-mortem to identify root cause
```

### Q9: What if someone tries to enumerate all short URLs?

**Answer:**

```
Attack: Attacker tries all possible 6-character combinations
Goal: Discover private/sensitive URLs

Calculation:
- 62^6 = 56.8 billion combinations
- At 1000 requests/second = 1.8 years to enumerate all
- At 100,000 requests/second = 6.5 days

Defenses:

1. Rate Limiting:
   - Per IP: 100 requests/minute
   - Per API key: Based on tier
   - Global: Circuit breaker at 50,000 QPS

2. Non-Sequential Short Codes:
   - Don't use sequential counter
   - Shuffle bits or add random offset
   - Makes prediction impossible

3. Honeypot URLs:
   - Create fake short URLs that trigger alerts
   - If accessed, flag the IP

4. CAPTCHA for Suspicious Patterns:
   - Detect enumeration attempts
   - Require CAPTCHA after threshold

5. Monitoring:
   - Alert on high 404 rates from single IP
   - Alert on sequential short code access patterns
```

### Q10: How do you handle URL expiration at scale?

**Answer:**

```
Challenge: 6 billion URLs, many with expiration dates
Need: Efficient cleanup without impacting production

Approach 1: Lazy Expiration (Chosen)
- Check expiration on read
- If expired, return 404
- Background job cleans up periodically

if (url.getExpiresAt() != null && url.getExpiresAt().isBefore(Instant.now())) {
    return ResponseEntity.status(410).body("URL has expired");
}

Approach 2: Scheduled Cleanup Job
- Run every hour
- Delete expired URLs in batches
- Use partitioned deletes

DELETE FROM urls
WHERE expires_at < NOW()
  AND expires_at > NOW() - INTERVAL '1 hour'
LIMIT 10000;

Approach 3: Database Partitioning
- Partition by expiration month
- Drop entire partitions when all URLs expired
- Most efficient for bulk cleanup

Why lazy expiration is primary:
- No immediate cleanup needed
- Reduces database write load
- Expired URLs don't hurt (just take space)
- Background job handles eventual cleanup
```

---

## Design Evolution Questions

### Q11: What would you build first with 2 weeks?

**Answer:**

```
MVP (2 weeks):

Week 1:
- Single PostgreSQL instance (no replication)
- Single Redis instance (no cluster)
- Basic API: create, redirect, delete
- Simple counter-based short code generation
- No analytics beyond click count

Week 2:
- Basic rate limiting (in-memory)
- Simple web UI for URL creation
- Health check endpoint
- Basic logging
- Deploy to single cloud instance

What's NOT included:
- Custom aliases
- User accounts
- Analytics dashboard
- CDN integration
- Kafka for events
- High availability

This handles ~1000 QPS, enough for initial launch.
```

### Q12: How does the design evolve from MVP to production?

**Answer:**

```
Phase 1: MVP (1K QPS)
├── Single PostgreSQL
├── Single Redis
├── 2 app servers
└── No CDN

Phase 2: Production (10K QPS)
├── PostgreSQL primary + 2 replicas
├── Redis Cluster (3 nodes)
├── 10 app servers with auto-scaling
├── CDN for edge caching
├── Kafka for analytics
└── Monitoring & alerting

Phase 3: Scale (100K QPS)
├── PostgreSQL sharded by short_code
├── Redis Cluster (30+ nodes)
├── 100+ app servers across regions
├── Multi-region CDN
├── ClickHouse for analytics
└── Global load balancing

Phase 4: Hyper-scale (1M+ QPS)
├── Custom storage engine
├── Edge computing for redirects
├── ML for abuse detection
├── Real-time analytics pipeline
└── Multi-cloud deployment
```

---

## Level-Specific Expectations

### L4 (Entry-Level) Expectations

Should demonstrate:

- Understanding of basic components (load balancer, cache, database)
- Simple capacity estimation
- Basic API design
- Awareness of caching benefits

May struggle with:

- Detailed failure scenarios
- Sharding strategies
- Advanced consistency models

### L5 (Mid-Level) Expectations

Should demonstrate:

- Complete end-to-end design
- Trade-off analysis for each decision
- Failure scenarios and mitigation
- Scaling strategies
- Production experience (monitoring, alerting)

May struggle with:

- Multi-region deployment
- Cost optimization at scale
- Advanced distributed systems concepts

### L6 (Senior) Expectations

Should demonstrate:

- Multiple solution approaches with trade-offs
- Deep dive into any component
- Cost analysis and optimization
- Evolution of design over time
- Handling edge cases (thundering herd, enumeration)
- Organizational considerations (team structure, on-call)

---

## Common Interviewer Pushbacks

### "Your latency target of 50ms seems aggressive"

**Response:**

```
You're right to question this. Let me break it down:

Cache hit path: 10-20ms
- CDN edge: 5ms
- Redis lookup: 1ms
- Response: 5ms

Cache miss path: 40-60ms
- CDN miss: 10ms
- Redis miss: 5ms
- PostgreSQL query: 10-20ms
- Response: 10ms

P50 should be cache hits (< 20ms)
P99 includes cache misses (< 100ms)

If 50ms is too aggressive, we can target:
- P50: 30ms
- P99: 150ms

This is still excellent UX for a redirect.
```

### "Why not use a NoSQL database for everything?"

**Response:**

```
I considered DynamoDB/Cassandra, but chose PostgreSQL because:

1. Analytics Requirements:
   - "Show me top 10 URLs by clicks this week"
   - "Group clicks by country and device"
   - These are natural in SQL, complex in NoSQL

2. Consistency Needs:
   - Custom alias uniqueness requires strong consistency
   - DynamoDB conditional writes work but are more complex

3. Operational Familiarity:
   - Team knows PostgreSQL well
   - Established backup/restore procedures
   - Easier to debug

If we had:
   - Pure key-value access patterns
   - No complex analytics
   - Need for automatic scaling

Then DynamoDB would be a better choice.
```

### "This seems over-engineered for a URL shortener"

**Response:**

```
Fair point. Let me clarify the scale we're designing for:

At 10B redirects/month:
- That's 3,858 QPS average, 12,000 peak
- This is Bitly-scale, not a hobby project

For a smaller scale (1M redirects/month):
- Single PostgreSQL instance
- Single Redis instance
- 2-3 app servers
- No Kafka, no CDN
- Cost: ~$500/month

The full architecture I described is for:
- 100:1 read-to-write ratio
- 99.99% availability requirement
- Sub-100ms latency globally
- Real-time analytics

I can simplify based on actual requirements.
```

---

## Summary: Key Points to Remember

1. **Read-heavy system**: 100:1 ratio, optimize for redirects
2. **Caching is critical**: Multi-layer (CDN → Redis → DB)
3. **Short code generation**: Counter + base62, or random with collision handling
4. **Analytics is async**: Kafka decouples redirect from tracking
5. **Consistency trade-off**: Strong for writes, eventual for reads acceptable
6. **Failure handling**: Every component has redundancy
7. **Security**: Rate limiting, input validation, no enumeration
8. **Evolution**: Start simple, scale as needed
