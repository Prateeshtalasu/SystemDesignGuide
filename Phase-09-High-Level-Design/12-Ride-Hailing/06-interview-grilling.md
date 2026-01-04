# Ride Hailing - Interview Grilling Q&A

## Trade-off Questions

### Q1: Why Redis for driver locations instead of a dedicated geospatial database?

**Answer:**

```
Redis is chosen because:

1. Sub-millisecond Latency: Finding nearby drivers in < 5ms is critical for 
   user experience. Redis geospatial commands (GEORADIUS) are extremely fast.

2. Built-in Expiry: Driver locations naturally expire. If a driver doesn't 
   update in 30 seconds, they're considered offline. Redis TTL handles this.

3. High Write Throughput: 100K location updates/second. Redis handles this 
   easily; traditional databases would struggle.

4. Simplicity: GEOADD and GEORADIUS are simple APIs. No complex query 
   optimization needed.

When would I choose a dedicated geospatial DB (PostGIS)?
- Complex polygon queries (surge zones)
- Historical analysis
- Joins with other data
- Persistence requirements

We actually use both: Redis for real-time, PostGIS for analytics.
```

### Q2: Why WebSocket instead of polling for real-time updates?

**Answer:**

```
WebSocket advantages:

1. Latency: Updates reach client in < 500ms vs 3-5 seconds with polling
2. Efficiency: No repeated HTTP overhead for each update
3. Bidirectional: Driver can send location AND receive ride requests
4. Battery: Less radio usage on mobile devices

Polling problems:
- 500K users polling every 3 seconds = 167K requests/second just for polling
- Most polls return "no update", wasting resources
- Noticeable delay in location updates

WebSocket challenges we handle:
- Connection management at scale (sticky sessions, reconnection)
- Cross-server messaging (Redis Pub/Sub)
- Mobile network transitions (reconnection logic)

For 700K concurrent connections, we use:
- NLB for Layer 4 load balancing
- Sticky sessions by connection ID
- Heartbeat every 30 seconds
- Exponential backoff on reconnection
```

### Q3: How do you handle the "celebrity problem" (viral ride request)?

**Answer:**

```
Scenario: A celebrity tweets "I'm taking an Uber from LAX!"
Result: Thousands of fake ride requests to that location

Defenses:

1. Rate Limiting:
   - Per user: 5 ride requests/minute
   - Per location: 100 requests/minute to same pickup
   - Per IP: Stricter limits for unauthenticated

2. Request Deduplication:
   - Same user can't have multiple active ride requests
   - Idempotency key prevents duplicate submissions

3. Geographic Rate Limiting:
   - Detect unusual request concentration
   - Temporarily increase surge pricing (discourages abuse)
   - Alert ops team for investigation

4. Account Verification:
   - Require phone verification
   - Flag accounts with suspicious patterns
   - Temporary holds on new accounts in affected area

5. Driver Protection:
   - Don't reveal exact pickup until driver accepts
   - Allow driver to cancel without penalty if suspicious
```

### Q4: Why partition Kafka by driver_id for location updates?

**Answer:**

```
Partitioning by driver_id ensures:

1. Ordering: All updates from one driver go to same partition, preserving 
   order. Important for velocity calculations and anomaly detection.

2. Stateful Processing: Consumer can maintain per-driver state (last known 
   location) without coordination.

3. Even Distribution: Hash of driver_id distributes evenly across partitions.

Alternative considered: Partition by geohash
- Pros: All updates in an area go to same partition
- Cons: Hot spots during events (concerts, sports games)
- Cons: Rebalancing needed as demand shifts

Why not partition by ride_id for ride events?
- We DO partition ride-events by ride_id
- Different topics have different partitioning strategies
- ride_id ensures all events for a ride are ordered
```

---

## Scaling Questions

### Q5: How would you handle 10x traffic increase?

**Answer:**

```
Current: 60K location updates/sec, 300 rides/sec
Target: 600K location updates/sec, 3K rides/sec

Changes needed:

1. Location Service:
   - Scale from 17 to 100+ pods
   - Increase Kafka partitions from 48 to 200
   - Add Redis cluster nodes (6 → 20)

2. WebSocket Layer:
   - Scale from 21 to 150+ servers
   - Implement connection limits per server
   - Add regional WebSocket clusters

3. Matching Service:
   - Scale from 5 to 30+ pods
   - Optimize matching algorithm (batch processing)
   - Pre-compute driver availability

4. Database:
   - Add read replicas (3 → 10)
   - Implement sharding by city/region
   - Use PgBouncer for connection pooling

5. Kafka:
   - Add brokers (6 → 20)
   - Increase partition count
   - Tune consumer parallelism

Cost implication:
- Current: ~$61K/month
- At 10x: ~$350K/month (not linear due to efficiencies)
```

### Q6: What's the first thing that breaks under load?

**Answer:**

```
Order of failure (based on experience):

1. WebSocket Connections (First)
   - Symptom: Connection refused, memory exhaustion
   - Cause: Each connection consumes ~10KB memory
   - Fix: Add servers, implement connection limits

2. Redis Memory
   - Symptom: Evictions, slow queries, OOM
   - Cause: Too many driver locations, ride states
   - Fix: Add nodes, increase memory, optimize data

3. Database Connections
   - Symptom: "too many connections" errors
   - Cause: Each pod opens connections, pool exhausted
   - Fix: PgBouncer, increase max_connections

4. Kafka Consumer Lag
   - Symptom: Analytics delayed, surge pricing stale
   - Cause: Consumers can't process fast enough
   - Fix: Add consumers, increase partitions

5. Maps API Quota
   - Symptom: 429 errors from Google
   - Cause: Exceeded daily/per-second quota
   - Fix: Caching, quota increase, fallback provider
```

### Q7: How does surge pricing work technically?

**Answer:**

```
Surge pricing balances supply and demand:

Calculation (every 60 seconds per zone):

1. Count ride requests in zone (last 5 minutes)
2. Count available drivers in zone
3. Calculate demand ratio = requests / drivers

Surge multiplier formula:
- ratio < 1.5: surge = 1.0 (no surge)
- ratio 1.5-3.0: surge = 1.0 + (ratio - 1.5) * 0.5
- ratio > 3.0: surge = 2.5 (capped)

Example:
- Zone has 100 requests in 5 minutes
- Zone has 40 available drivers
- Ratio = 100/40 = 2.5
- Surge = 1.0 + (2.5 - 1.5) * 0.5 = 1.5x

Storage:
- Surge values stored in Redis with 5-minute TTL
- Key: surge:{zone_id}, Value: "1.5"

Challenges:
1. Zone boundaries: Driver on edge might see different surge
   Solution: Smooth transitions, use pickup location

2. Rapid changes: Surge shouldn't flip-flop
   Solution: Gradual changes, minimum duration (5 min)

3. Driver gaming: Drivers wait for surge
   Solution: Show surge zones to riders, not drivers
```

---

## Failure Scenarios

### Q8: What happens if a driver's app crashes mid-trip?

**Answer:**

```
Scenario: Driver's phone dies during active ride

Detection:
1. No location updates for 60 seconds
2. WebSocket heartbeat fails
3. System marks driver as "disconnected"

Immediate actions:
1. Rider notified: "Driver connection lost"
2. Trip continues in "degraded" mode
3. Last known location shown to rider

Recovery paths:

Path A: Driver reconnects within 5 minutes
- App syncs state from server
- Trip resumes normally
- No action needed

Path B: Driver doesn't reconnect
- After 5 minutes, rider prompted: "End trip here?"
- If yes: Fare calculated to last known location
- If no: Support ticket created

Path C: Rider ends trip manually
- Fare calculated based on actual distance traveled
- Driver's account flagged for review

Safety measures:
- Rider can share trip with contacts
- Emergency button always visible
- Support can call driver's phone
```

### Q9: How do you prevent payment fraud?

**Answer:**

```
Fraud vectors and mitigations:

1. Stolen Credit Cards:
   - Address verification (AVS)
   - CVV required
   - 3D Secure for high-value rides
   - Velocity limits (max rides/day per card)

2. Fake Rides (driver-rider collusion):
   - GPS verification of actual travel
   - Compare claimed route vs actual
   - Flag rides with same driver-rider pairs
   - Machine learning on ride patterns

3. Fare Manipulation:
   - Server-side fare calculation only
   - Route verification against Maps API
   - Audit trail of all fare components

4. Promo Code Abuse:
   - One-time use codes
   - Device fingerprinting
   - Phone number verification
   - Velocity limits per account

5. Chargeback Fraud:
   - Detailed trip records with GPS
   - Driver/rider photos
   - Signed receipts for cash rides
   - Dispute resolution process

Detection system:
- Real-time scoring of each transaction
- ML model trained on historical fraud
- Manual review queue for flagged rides
```

### Q10: What if the matching service goes down?

**Answer:**

```
Matching service is critical - no matches = no rides

Failure modes:

1. Single pod crash:
   - Kubernetes restarts pod automatically
   - Other pods handle load
   - Impact: < 1 second, minimal

2. All pods crash:
   - Symptom: Ride requests timeout
   - Detection: Health checks fail, alerts fire
   - Recovery: Auto-scaling brings up new pods (2-3 min)

3. Redis dependency failure:
   - Circuit breaker opens
   - Fallback to PostGIS queries
   - Latency increases from 5ms to 50ms
   - System degraded but functional

Mitigation strategies:

1. Multi-AZ deployment:
   - Pods spread across availability zones
   - Single AZ failure doesn't take down service

2. Circuit breakers on all dependencies:
   - Redis, PostgreSQL, Maps API
   - Graceful degradation vs hard failure

3. Request queuing:
   - If matching slow, queue requests
   - Process when capacity available
   - Better than immediate failure

4. Cached driver locations:
   - Keep 30-second cache of driver positions
   - Can match even if Redis temporarily down

5. Manual override:
   - Admin can manually assign drivers
   - Used during extended outages
```

---

## Design Evolution Questions

### Q11: What would you build first with 4 weeks?

**Answer:**

```
MVP (4 weeks):

Week 1-2: Core Ride Flow
- User registration (phone verification)
- Basic ride request
- Simple matching (nearest driver)
- Driver accept/decline
- Trip status updates

Week 3: Payment & Completion
- Stripe integration
- Fare calculation (distance + time)
- Trip completion
- Basic ratings

Week 4: Polish & Deploy
- Basic admin dashboard
- Error handling
- Monitoring setup
- Production deployment

What's NOT included:
- Surge pricing
- Scheduled rides
- Ride pooling
- Advanced matching algorithm
- Real-time ETA updates
- Push notifications
- Driver earnings dashboard

This MVP handles ~1000 rides/day with basic functionality.
```

### Q12: How does the design evolve from MVP to production?

**Answer:**

```
Phase 1: MVP (1K rides/day)
├── Single PostgreSQL
├── Single Redis
├── 3 app servers
├── Basic matching (nearest driver)
└── Simple fare calculation

Phase 2: Production (50K rides/day)
├── PostgreSQL primary + 2 replicas
├── Redis Cluster (6 nodes)
├── 20 app servers
├── WebSocket for real-time
├── Kafka for events
├── Surge pricing
├── Push notifications
└── Monitoring & alerting

Phase 3: Scale (500K rides/day)
├── Multi-region deployment
├── 100+ app servers
├── Advanced matching (ML-based)
├── Ride pooling
├── Real-time ETA (ML model)
├── Fraud detection
└── A/B testing infrastructure

Phase 4: Global (5M rides/day)
├── Per-city deployments
├── Custom routing engine
├── Predictive demand modeling
├── Dynamic pricing optimization
├── Driver incentive systems
└── Autonomous vehicle integration
```

---

## Level-Specific Expectations

### L4 (Entry-Level) Expectations

Should demonstrate:
- Understanding of basic components (API, database, cache)
- Simple matching algorithm (nearest driver)
- Basic capacity estimation
- Awareness of real-time requirements

May struggle with:
- Geospatial indexing details
- WebSocket scaling
- Surge pricing implementation
- Failure scenarios

### L5 (Mid-Level) Expectations

Should demonstrate:
- Complete end-to-end design
- Geospatial queries with Redis
- WebSocket architecture
- Event-driven design with Kafka
- Failure scenarios and mitigation
- Monitoring strategy

May struggle with:
- Multi-region deployment
- Advanced matching algorithms
- Cost optimization at scale
- ML integration for ETA

### L6 (Senior) Expectations

Should demonstrate:
- Multiple architecture approaches with trade-offs
- Deep dive into any component
- Cost analysis and optimization
- Evolution of design over time
- Handling edge cases (surge, fraud, failures)
- Organizational considerations (team structure)
- Integration with broader ecosystem

---

## Common Interviewer Pushbacks

### "Your matching algorithm seems too simple"

**Response:**

```
You're right that nearest-driver is simplistic. In production:

Factors to consider:
1. Distance (40% weight) - ETA to pickup
2. Driver rating (20% weight) - Quality
3. Acceptance rate (15% weight) - Reliability
4. Vehicle match (15% weight) - Type requested
5. Driver earnings (10% weight) - Fairness

Advanced approaches:
1. ML-based matching:
   - Train on historical completion rates
   - Predict acceptance probability
   - Optimize for completion, not just distance

2. Batch matching:
   - Collect requests over 2-3 seconds
   - Solve assignment problem globally
   - Better overall efficiency

3. Predictive positioning:
   - Predict demand by time/location
   - Incentivize drivers to position optimally
   - Reduce average pickup time

For this interview, I focused on the simpler version to cover 
more ground. Happy to deep dive on matching if you'd like.
```

### "How do you handle drivers in rural areas?"

**Response:**

```
Rural areas have unique challenges:

1. Sparse Driver Coverage:
   - Expand search radius (5km → 20km)
   - Show estimated wait time honestly
   - Allow scheduled rides (driver can plan)

2. Longer Trips:
   - Adjust pricing for long-distance
   - Ensure driver gets return fare or next ride
   - Partner with local drivers

3. Poor Connectivity:
   - Offline mode for drivers
   - Sync when connection available
   - SMS fallback for notifications

4. Different Economics:
   - Lower commission in rural areas
   - Incentive programs for rural drivers
   - Partner with local taxi companies

5. Alternative Solutions:
   - Scheduled rides only (no on-demand)
   - Carpooling with regular commuters
   - Integration with public transit
```

### "This seems expensive for a startup"

**Response:**

```
You're right. Let me break down costs by stage:

Startup (1K rides/day):
- Single server: $200/month
- Managed DB: $100/month
- Redis: $50/month
- Maps API: $500/month
- Total: ~$850/month

Growth (10K rides/day):
- 5 servers: $1,000/month
- RDS: $500/month
- ElastiCache: $200/month
- Maps API: $2,000/month
- Total: ~$4,000/month

Scale (100K rides/day):
- 30 servers: $6,000/month
- RDS Multi-AZ: $3,000/month
- Redis Cluster: $1,000/month
- Kafka: $2,000/month
- Maps API: $10,000/month
- Total: ~$25,000/month

The full $61K/month architecture I described is for 
5M rides/day. A startup would use a much simpler setup.

Key insight: Infrastructure cost is tiny compared to 
driver payments, marketing, and support costs.
```

---

## Summary: Key Points to Remember

1. **Real-time is critical**: WebSocket, Redis, sub-second updates
2. **Geospatial at scale**: Redis GEORADIUS for < 5ms queries
3. **Event-driven**: Kafka decouples services, enables analytics
4. **Matching complexity**: Balance ETA, quality, fairness
5. **Surge pricing**: Supply/demand balancing, zone-based
6. **Failure handling**: Circuit breakers, graceful degradation
7. **Security**: Location verification, fraud prevention
8. **Cost awareness**: Maps API often largest cost
9. **Evolution**: Start simple, scale as needed

