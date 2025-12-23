# Pastebin - Interview Grilling Q&A

## Trade-off Questions

### Q1: Why separate PostgreSQL and S3 instead of storing content in the database?

**Answer:**
```
Separation provides several benefits:

1. Cost Efficiency:
   - S3: $0.023/GB/month
   - RDS: $0.115/GB/month (5x more expensive)
   - At 12 TB/year, savings: $1,100/month

2. Performance:
   - Large TEXT columns slow down queries
   - Index scans skip content entirely
   - Backup/restore is faster

3. Scalability:
   - S3 scales infinitely
   - Database stays small and fast
   - CDN integrates directly with S3

4. Operational:
   - Database backups are smaller
   - Can use S3 lifecycle policies
   - Easier disaster recovery

When would I use database storage?
   - Very small pastes (< 1KB)
   - Need transactional consistency
   - Simpler architecture for MVP
```

### Q2: How do you handle very large pastes (10 MB)?

**Answer:**
```
Large paste handling:

1. Upload Path:
   - Use multipart/form-data
   - Stream directly to S3 (don't buffer in memory)
   - Set timeout appropriately (30+ seconds)

2. Download Path:
   - Skip Redis cache (too large)
   - Stream from S3 through CDN
   - Use Range headers for partial downloads

3. Memory Management:
   - Never load full content in memory
   - Use streaming APIs
   - Set appropriate JVM heap limits

4. Rate Limiting:
   - Stricter limits for large pastes
   - Count against storage quota

Code example:
```java
public void streamLargePaste(String key, OutputStream out) {
    s3Client.getObject(
        GetObjectRequest.builder().bucket(bucket).key(key).build(),
        ResponseTransformer.toOutputStream(out)
    );
}
```

### Q3: Why not use MongoDB for everything?

**Answer:**
```
MongoDB could work, but PostgreSQL + S3 is better because:

1. Query Flexibility:
   - Complex queries for analytics (GROUP BY, JOINs)
   - PostgreSQL has mature query optimizer
   - MongoDB aggregation is more complex

2. Cost:
   - MongoDB Atlas storage is expensive
   - S3 is 10x cheaper for content

3. Consistency:
   - PostgreSQL has strong ACID guarantees
   - Easier to reason about transactions

4. Operational Experience:
   - Most teams know PostgreSQL well
   - Better tooling ecosystem

When MongoDB makes sense:
   - Schema-less requirements
   - Document-oriented data model
   - Already using MongoDB elsewhere
```

---

## Scaling Questions

### Q4: How would you handle 100x traffic?

**Answer:**
```
Current: 200 QPS peak
Target: 20,000 QPS peak

Changes needed:

1. CDN Optimization:
   - Increase cache TTL
   - Add more edge locations
   - Target 90%+ cache hit rate

2. Application Layer:
   - Scale from 5 to 50+ pods
   - Add read-through caching
   - Optimize hot paths

3. Database:
   - Add read replicas (5-10)
   - Connection pooling with PgBouncer
   - Consider read/write splitting

4. Storage:
   - S3 handles scale automatically
   - Add S3 Transfer Acceleration
   - Multi-region replication

5. Cache:
   - Scale Redis cluster
   - Increase memory allocation
   - Add local caching layer

Cost at 100x:
   - Current: ~$15,000/month
   - At 100x: ~$100,000/month (mostly bandwidth)
```

### Q5: What breaks first under load?

**Answer:**
```
Bottleneck analysis:

1. Database Connections (First to break)
   - Default PostgreSQL: 100 connections
   - 50 pods × 20 connections = 1000 needed
   - Solution: PgBouncer connection pooling

2. S3 Request Rate
   - S3 has per-prefix limits (3,500 PUT, 5,500 GET/sec)
   - Solution: Randomize key prefixes

3. Redis Memory
   - Hot pastes exceed cache capacity
   - Solution: Tiered caching, LRU eviction

4. Bandwidth
   - CDN egress costs spike
   - Solution: Aggressive compression, longer TTLs

5. Cleanup Worker
   - Falls behind on expired pastes
   - Solution: Parallel workers, larger batches
```

---

## Failure Scenarios

### Q6: What if S3 becomes unavailable?

**Answer:**
```
S3 is highly available (99.99%), but if it fails:

Impact:
- Cannot create new pastes (content storage fails)
- Cannot view pastes not in cache

Mitigation:

1. Multi-Region Replication:
   - Replicate to secondary region
   - Failover automatically

2. Cache as Buffer:
   - Redis holds recent pastes
   - Serve from cache during outage

3. Graceful Degradation:
   - Accept paste creation, queue for later upload
   - Show "temporarily unavailable" for uncached

4. Circuit Breaker:
   - Stop trying after N failures
   - Return 503 quickly

Recovery:
- S3 auto-recovers
- Process queued uploads
- Clear any stale cache entries
```

### Q7: How do you prevent abuse (spam, malware)?

**Answer:**
```
Multi-layer defense:

1. Rate Limiting:
   - Per IP: 10 pastes/hour (anonymous)
   - Per user: 100 pastes/hour
   - Global: 1000 pastes/minute

2. Content Scanning:
   - Check for known malware signatures
   - Detect phishing patterns
   - Flag suspicious URLs

3. CAPTCHA:
   - Required for anonymous users
   - Triggered after suspicious activity

4. Reporting System:
   - Users can report malicious content
   - Auto-hide after N reports

5. Honeypot Detection:
   - Hidden form fields
   - Timing analysis

6. Account Verification:
   - Email verification for accounts
   - Phone verification for high limits

Response to abuse:
- Immediate: Block IP, delete content
- Short-term: Increase rate limits
- Long-term: Improve detection models
```

### Q8: What happens when a paste goes viral?

**Answer:**
```
Scenario: Paste gets 1M views in 1 hour

Problems:
1. CDN cache might be cold
2. All requests hit origin
3. S3 throttling possible

Solutions:

1. CDN Pre-warming:
   - Detect viral potential early
   - Push to all edge locations

2. Request Coalescing:
   - Multiple requests wait for one S3 fetch
   - Prevents thundering herd

3. Local Cache:
   - Cache in application memory
   - Even 10-second TTL helps

4. S3 Request Spreading:
   - Use multiple object keys
   - Round-robin between copies

5. Graceful Degradation:
   - Serve plain text if highlighting fails
   - Queue analytics, don't block response
```

---

## Design Evolution

### Q9: What would you build in 2 weeks (MVP)?

**Answer:**
```
MVP Scope:

Week 1:
- Single PostgreSQL (content in DB)
- Basic API: create, view, delete
- Simple web UI
- No user accounts

Week 2:
- Basic rate limiting
- Syntax highlighting (client-side)
- Expiration (manual cleanup)
- Deploy to single server

NOT included:
- S3 storage (use DB)
- Redis caching
- CDN
- User accounts
- Password protection
- Analytics

This handles ~100 QPS, enough for launch.
```

### Q10: How does the design evolve?

**Answer:**
```
Phase 1: MVP (100 QPS)
├── Single PostgreSQL (content in TEXT column)
├── No caching
├── Single server
└── Basic expiration

Phase 2: Production (1K QPS)
├── Move content to S3
├── Add Redis caching
├── Add CDN (CloudFront)
├── User accounts
├── Multiple app servers
└── Automated cleanup

Phase 3: Scale (10K QPS)
├── PostgreSQL read replicas
├── Redis cluster
├── Multi-region CDN
├── Content deduplication
├── Advanced rate limiting
└── Analytics pipeline

Phase 4: Enterprise (100K+ QPS)
├── Database sharding
├── Multi-region deployment
├── Custom CDN rules
├── ML-based abuse detection
└── Enterprise features (teams, SSO)
```

---

## Level-Specific Expectations

### L4 (Entry-Level)

Should demonstrate:
- Basic understanding of web architecture
- Simple database design
- Awareness of caching benefits
- Basic API design

May struggle with:
- Storage optimization
- Scaling strategies
- Security considerations

### L5 (Mid-Level)

Should demonstrate:
- Complete end-to-end design
- Trade-off analysis
- Failure scenarios
- Production considerations
- Cost awareness

May struggle with:
- Multi-region deployment
- Complex caching strategies
- Advanced security

### L6 (Senior)

Should demonstrate:
- Multiple solution approaches
- Deep dive into any component
- Cost optimization
- Organizational considerations
- Edge cases (viral content, abuse)

---

## Common Pushbacks

### "Why not just use a CDN for everything?"

**Response:**
```
CDN limitations:
1. Can't handle dynamic content (private pastes)
2. Cache invalidation is slow (~15 min)
3. No access control at edge
4. Origin still needed for writes

CDN is great for:
- Public, static content
- Geographic distribution
- DDoS protection

We use CDN for raw content, but need origin for:
- Authentication
- Rate limiting
- Dynamic metadata
```

### "This seems over-engineered"

**Response:**
```
The design scales with requirements:

For 1M pastes/month:
- Single PostgreSQL
- Content in database
- Simple caching
- Cost: ~$500/month

For 10M pastes/month (our target):
- Need S3 for cost efficiency
- Need CDN for global access
- Need Redis for performance
- Cost: ~$15,000/month

I can simplify based on actual scale.
```

---

## Summary: Key Points

1. **Storage is the challenge**: Unlike URL Shortener, Pastebin is storage-heavy
2. **Separate metadata and content**: PostgreSQL + S3 is optimal
3. **Compression matters**: 70-80% storage savings
4. **CDN is essential**: Reduces origin load and latency
5. **Cleanup is critical**: Expired pastes accumulate fast
6. **Security layers**: Rate limiting, content scanning, access control
7. **Start simple**: MVP can use database-only, add S3 later

