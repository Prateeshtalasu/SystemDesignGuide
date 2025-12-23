# Typeahead / Autocomplete - Interview Grilling Q&A

## Trade-off Questions

### Q1: Why Trie over other data structures?

**Answer:**
```
Trie is optimal for prefix matching:

Comparison:
┌─────────────────┬────────────────┬─────────────────┐
│ Structure       │ Prefix Search  │ Space           │
├─────────────────┼────────────────┼─────────────────┤
│ Sorted Array    │ O(log N × M)   │ O(N × M)        │
│ Hash Map        │ O(N × M)       │ O(N × M)        │
│ Trie            │ O(M)           │ O(N × M) shared │
│ Ternary Search  │ O(M + log N)   │ O(N × M)        │
└─────────────────┴────────────────┴─────────────────┘

N = number of queries (5 billion)
M = average query length (20 chars)

Trie advantages:
1. O(M) lookup regardless of N
2. Prefix sharing reduces memory
3. Natural structure for suggestions

Alternatives considered:
- Elasticsearch: Good but adds network hop
- Redis sorted sets: Can't do prefix efficiently
- Bloom filters: Only membership, no retrieval
```

### Q2: Why pre-compute suggestions instead of computing at query time?

**Answer:**
```
Pre-computation is essential for latency:

At query time (without pre-computation):
1. Find node for prefix: O(M) ✓
2. Collect all completions: O(K) where K = matching queries
3. Sort by score: O(K log K)
4. Return top 10: O(1)

For prefix "how", K could be millions!
Total: O(M + K log K) = too slow

With pre-computation:
1. Find node for prefix: O(M)
2. Return node.topSuggestions: O(1)
Total: O(M) = ~20 operations = microseconds

Trade-off:
- More memory (store suggestions at each node)
- Slower index building
- But: O(1) query time is worth it

Memory cost:
- 500M prefix nodes × 15 suggestions × 8 bytes = 60 GB
- Acceptable for our 400 GB memory budget
```

### Q3: Why hourly updates instead of real-time?

**Answer:**
```
Hourly updates balance freshness vs complexity:

Real-time updates would require:
1. Stream processing of every search
2. Distributed Trie updates
3. Consistency across 60 servers
4. Much more complex architecture

Hourly updates:
1. Batch processing is simpler
2. Atomic index swaps
3. Consistent view across servers
4. 1 hour staleness is acceptable

When real-time matters:
- Breaking news: "earthquake california"
- Viral events: "super bowl halftime"

Solution for trending:
- Separate "trending" module updated every 5 min
- Merge trending into suggestions at query time
- Best of both worlds

Most queries are evergreen:
- "how to tie a tie" doesn't change hourly
- 95% of suggestions are stable
```

---

## Scaling Questions

### Q4: How would you handle 10x traffic?

**Answer:**
```
Current: 500K QPS
Target: 5M QPS

Scaling approach:

1. Horizontal scaling (primary):
   - Add more suggestion servers
   - 60 servers → 600 servers
   - Each server handles same Trie

2. CDN optimization:
   - Increase cache TTL
   - Pre-warm popular prefixes
   - Target 60% cache hit rate (vs 40%)

3. Regional deployment:
   - Deploy in more regions
   - Reduce cross-region traffic
   - Local Trie copies

4. Sharding (if needed):
   - Shard by first character
   - 26 shards for a-z
   - Each shard smaller, faster

Cost at 10x:
- Servers: 600 × $3000/month = $1.8M/month
- Bandwidth: 10x = $4M/month
- Total: ~$6M/month

Optimization:
- Reserved instances: 40% savings
- Spot for non-critical: 70% savings
- Optimized: ~$4M/month
```

### Q5: What breaks first under load?

**Answer:**
```
Bottleneck analysis:

1. Network bandwidth (First to break)
   - 5M QPS × 600 bytes = 3 GB/s
   - Need 24+ Gbps per region
   - Solution: Add more load balancers, CDN

2. CPU on suggestion servers
   - Trie lookup is memory-bound, not CPU
   - But serialization is CPU-intensive
   - Solution: Use efficient serialization (protobuf)

3. Memory pressure
   - More requests = more object allocation
   - GC pauses increase
   - Solution: Object pooling, tune GC

4. CDN capacity
   - Edge locations overwhelmed
   - Solution: Add edge locations, increase cache

5. Pipeline throughput
   - More logs to process
   - Solution: Scale Spark cluster
```

### Q6: How do you handle a trending topic explosion?

**Answer:**
```
Scenario: Celebrity death, 10M searches/minute for their name

Problems:
1. CDN cache is cold for this query
2. All requests hit origin
3. Query not in current Trie (too new)

Solutions:

1. Fast-path for trending:
   - Separate trending service (5-min updates)
   - Merge at query time
   
   if (trendingService.isTrending(prefix)) {
       return mergeSuggestions(
           trie.getSuggestions(prefix),
           trendingService.getSuggestions(prefix)
       );
   }

2. CDN cache warming:
   - Detect trending queries
   - Push to CDN edges proactively

3. Graceful degradation:
   - If overloaded, return cached/stale results
   - "No suggestions" is acceptable briefly

4. Auto-scaling:
   - Trigger scale-up on traffic spike
   - Add servers in minutes
```

---

## Failure Scenarios

### Q7: What if the data pipeline fails?

**Answer:**
```
Scenario: Spark job fails, no new index for 6 hours

Impact:
- Suggestions become stale
- New trending topics missing
- No data loss (servers keep old index)

Mitigation:

1. Monitoring:
   - Alert if pipeline doesn't complete
   - Alert if index age > 2 hours

2. Redundancy:
   - Run pipeline in multiple regions
   - Cross-region failover

3. Manual intervention:
   - On-call can trigger manual build
   - Can roll back to previous index

4. Graceful handling:
   - Servers continue with old index
   - 6-hour-old suggestions still useful
   - Most queries are evergreen

Recovery:
1. Fix pipeline issue
2. Run catch-up job
3. Deploy new index
4. Verify suggestions quality
```

### Q8: What if a server runs out of memory?

**Answer:**
```
Scenario: Server OOM during index load

Causes:
1. Index too large
2. Memory leak
3. GC not keeping up

Prevention:

1. Memory limits:
   - Set JVM heap < physical RAM
   - Leave room for OS and buffers
   - -Xmx450g for 512GB server

2. Index size monitoring:
   - Alert if index grows unexpectedly
   - Fail build if > threshold

3. Staged rollout:
   - Load new index on 1 server first
   - Monitor memory
   - Roll out to others

4. Circuit breaker:
   - If memory > 90%, stop accepting requests
   - Shed load to healthy servers

Recovery:
1. LB removes unhealthy server
2. Pod restarts automatically
3. Downloads current index
4. Rejoins serving pool
```

### Q9: How do you prevent offensive suggestions?

**Answer:**
```
Multi-layer filtering:

1. Pre-processing (pipeline):
   - Blocklist of known bad queries
   - Profanity detection
   - PII patterns (emails, phones)
   - Never added to Trie

2. Real-time filtering:
   - Check suggestions before returning
   - Emergency blocklist (updated instantly)
   - Regex patterns for new threats

3. Human review:
   - Sample suggestions for quality
   - User reports of bad suggestions
   - Regular audits

4. Reactive measures:
   - Instant blocklist update via Zookeeper
   - All servers pick up within seconds
   - No index rebuild needed

Example flow:
1. User reports "offensive suggestion X"
2. Moderator adds to blocklist
3. Zookeeper updated
4. All servers block within 10 seconds
5. Post-incident: add to pipeline blocklist
```

---

## Design Evolution

### Q10: What would you build in 2 weeks (MVP)?

**Answer:**
```
MVP Scope:

Week 1:
- Simple Trie in memory
- Load from static file (not pipeline)
- Basic API endpoint
- No personalization
- No trending

Week 2:
- Simple scoring (frequency only)
- Basic content filtering
- Deploy to 2 servers
- Simple load balancer

NOT included:
- Data pipeline (use static data)
- CDN caching
- Personalization
- Trending detection
- Multi-region

Scale: ~10K QPS
Latency: ~50ms
Cost: ~$2,000/month

Good enough to validate the feature.
```

### Q11: How does the design evolve?

**Answer:**
```
Phase 1: MVP (10K QPS)
├── Static Trie from file
├── 2 servers
├── No pipeline
└── ~$2K/month

Phase 2: Production (100K QPS)
├── Hourly pipeline (Spark)
├── 10 servers
├── CDN caching
├── Basic ranking
└── ~$30K/month

Phase 3: Scale (500K QPS)
├── Full pipeline
├── 60 servers
├── Multi-region
├── Personalization
├── Trending detection
└── ~$400K/month

Phase 4: Global (5M QPS)
├── Sharded Trie
├── 600+ servers
├── Edge computing
├── ML-based ranking
├── Real-time trending
└── ~$4M/month
```

---

## Level-Specific Expectations

### L4 (Entry-Level)

Should demonstrate:
- Understanding of Trie data structure
- Basic API design
- Awareness of latency requirements
- Simple capacity estimation

May struggle with:
- Pre-computation optimization
- Pipeline design
- Multi-region deployment

### L5 (Mid-Level)

Should demonstrate:
- Complete Trie with pre-computed suggestions
- Data pipeline architecture
- Caching strategy
- Failure scenarios
- Production considerations

May struggle with:
- Complex ranking algorithms
- Real-time trending
- Cost optimization at scale

### L6 (Senior)

Should demonstrate:
- Multiple approaches with trade-offs
- Deep dive into any component
- ML-based ranking considerations
- Organizational impact (team size, ops)
- Cost analysis and optimization

---

## Common Pushbacks

### "Why not use Elasticsearch?"

**Response:**
```
Elasticsearch is powerful but adds latency:

With Elasticsearch:
  Client → CDN → LB → App → Elasticsearch → Response
  Minimum: 5 + 1 + 5 + 5 = 16ms

With in-memory Trie:
  Client → CDN → LB → App (Trie) → Response
  Minimum: 5 + 1 + 1 = 7ms

For typeahead, every millisecond matters.

Elasticsearch makes sense when:
- Need full-text search (not just prefix)
- Need complex queries
- Can tolerate 20-50ms latency
- Don't want to manage Trie infrastructure

Our choice: Trie for speed, accept complexity.
```

### "500K QPS seems unrealistic"

**Response:**
```
Let me break down the math:

500M daily active users
× 5 searches per user per day
× 8 keystroke requests per search
= 20 billion requests/day

20B / 86,400 seconds = 231K QPS average

Peak is 3x average = ~700K QPS

I rounded to 500K for design.

For comparison:
- Google: Likely 10x this
- Bing: Similar scale
- E-commerce search: 10-50K QPS

This is a large-scale system, but realistic
for a major search engine.
```

### "This is expensive ($400K/month)"

**Response:**
```
Cost per query: $0.67 per million queries

For context:
- 600 billion queries/month
- $400K / 600B = $0.00000067 per query

Revenue consideration:
- If 1% of queries lead to ad click
- Ad click worth $0.10
- Revenue: 600B × 0.01 × $0.10 = $600M/month

ROI is excellent.

Cost optimization opportunities:
1. Reserved instances: -40%
2. Spot for pipeline: -70%
3. Better compression: -20% bandwidth
4. Longer cache TTL: -30% origin traffic

Optimized: ~$250K/month
```

---

## Summary: Key Points

1. **Trie is essential**: O(prefix length) lookup is unbeatable
2. **Pre-compute suggestions**: Don't compute at query time
3. **Memory is the constraint**: 400GB per server, 60 servers
4. **Latency is critical**: < 50ms or users notice
5. **Batch updates are fine**: Hourly is good enough
6. **CDN helps but limited**: 40% hit rate for popular prefixes
7. **Personalization is a boost**: Not core functionality
8. **Content filtering is mandatory**: Legal and brand safety

