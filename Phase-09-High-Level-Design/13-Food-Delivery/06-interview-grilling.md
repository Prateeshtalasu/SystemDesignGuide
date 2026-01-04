# Food Delivery - Interview Grilling Q&A

## Trade-off Questions

### Q1: Why Elasticsearch instead of PostgreSQL full-text search?

**Answer:**

```
Elasticsearch is chosen because:

1. Performance at Scale:
   - 200K restaurants, 20M menu items
   - Complex queries: text + geo + filters
   - PostgreSQL FTS degrades with complex queries
   - ES designed for this workload

2. Relevance Tuning:
   - Custom scoring (rating, distance, promotions)
   - Boosting certain fields
   - Synonym handling
   - Autocomplete support

3. Geo Queries:
   - Native geo_point type
   - Efficient radius searches
   - Distance decay scoring
   - Bounding box filters

4. Horizontal Scaling:
   - Add nodes as data grows
   - Sharding is automatic
   - PostgreSQL scaling is harder

When would I choose PostgreSQL FTS?
- Smaller dataset (< 10K restaurants)
- Simpler queries (no geo, no custom scoring)
- Tight budget (avoid ES cluster costs)
- Strong consistency requirements
```

### Q2: Why cache menus for only 15 minutes?

**Answer:**

```
15-minute TTL balances freshness and performance:

Why short TTL:
1. Items can sell out quickly (popular dishes)
2. Prices can change (promotions, surge pricing)
3. Availability changes (kitchen closed certain items)
4. Customers expect accurate info

Why not shorter (e.g., 1 minute):
1. Menu data is expensive to load (50KB average)
2. Would overwhelm database during peak
3. Most changes don't need instant propagation
4. Item availability verified at order time anyway

Why not longer (e.g., 1 hour):
1. Customer sees "available" but item is sold out
2. Price changes not reflected
3. Bad user experience leads to abandoned carts

Invalidation strategy:
- Explicit invalidation on menu item update
- 15-minute TTL is backup
- Critical items (availability) checked at order time
```

### Q3: How do you handle a restaurant overwhelmed with orders?

**Answer:**

```
Multiple strategies to protect restaurants:

1. Order Pacing:
   - Track orders per restaurant per hour
   - If > threshold, increase ETA shown to customers
   - Customers self-select away

2. Automatic Pause:
   - Restaurant can set max orders/hour
   - System auto-pauses when limit reached
   - Resumes after cooldown period

3. Manual Controls:
   - Restaurant can pause orders anytime
   - "Busy" mode: longer ETAs, no promotions
   - "Closed" mode: no new orders

4. Smart Routing:
   - Show overwhelmed restaurants lower in search
   - Boost similar alternatives
   - "Busy" badge warns customers

Implementation:
```java
public boolean canAcceptOrder(String restaurantId) {
    // Check if paused
    if (restaurantService.isPaused(restaurantId)) {
        return false;
    }
    
    // Check order rate
    int recentOrders = orderRepository
        .countByRestaurantInLastHour(restaurantId);
    int maxOrders = restaurantService.getMaxOrdersPerHour(restaurantId);
    
    if (recentOrders >= maxOrders) {
        // Auto-pause and notify
        restaurantService.autoPause(restaurantId, Duration.ofMinutes(30));
        return false;
    }
    
    return true;
}
```

### Q4: Why use the outbox pattern for order events?

**Answer:**

```
The outbox pattern solves dual-write problem:

Problem without outbox:
1. Save order to PostgreSQL
2. Publish event to Kafka
3. If step 2 fails, order exists but no notification
4. If step 1 fails after step 2, event exists but no order

With outbox pattern:
1. Save order + outbox event in same transaction
2. Separate process publishes events to Kafka
3. If Kafka fails, events stay in outbox
4. Eventually consistent, but never lost

```java
@Transactional
public Order createOrder(OrderRequest request) {
    // Both in same transaction
    Order order = orderRepository.save(buildOrder(request));
    outboxRepository.save(new OutboxEvent(order.getId(), "ORDER_PLACED", order));
    return order;
}
```

Alternatives considered:

1. Kafka Transactions:
   - Requires Kafka-only consumers
   - More complex setup
   - Less flexible

2. Saga Pattern:
   - For multi-service transactions
   - Overkill for single service
   - More complex

3. Event Sourcing:
   - Events are source of truth
   - Requires major architecture change
   - Not needed for our use case
```

---

## Scaling Questions

### Q5: How would you handle 10x traffic (Super Bowl Sunday)?

**Answer:**

```
Current: 30 orders/second peak
Target: 300 orders/second

Pre-event preparation:

1. Capacity Planning:
   - Pre-scale all services 3x
   - Warm up caches with popular restaurants
   - Pre-provision database connections

2. Search Optimization:
   - Increase Elasticsearch replicas
   - Pre-cache popular search queries
   - Reduce search result size

3. Order Processing:
   - Scale Order Service pods
   - Increase Kafka partitions
   - Add more consumers

4. Database:
   - Add read replicas
   - Enable connection pooling
   - Pre-allocate IDs

5. Graceful Degradation:
   - Disable recommendations
   - Simplify search ranking
   - Reduce real-time updates

Real-time during event:

1. Monitor dashboards
2. Auto-scaling triggers
3. Manual intervention if needed
4. Communication with restaurants

Cost: ~$50K for one day (vs $900/day normal)
```

### Q6: What's the first thing that breaks under load?

**Answer:**

```
Order of failure during peak:

1. Search Service (First)
   - Symptom: Slow restaurant discovery
   - Cause: Elasticsearch overwhelmed
   - Impact: Customers can't find restaurants
   - Fix: Add ES nodes, enable caching

2. Menu Cache
   - Symptom: Cache misses spike
   - Cause: Too many unique restaurants accessed
   - Impact: Database overloaded
   - Fix: Increase Redis memory, extend TTL

3. Database Connections
   - Symptom: Connection pool exhausted
   - Cause: Too many concurrent orders
   - Impact: Orders fail
   - Fix: PgBouncer, increase pool size

4. Payment Processing
   - Symptom: Payment timeouts
   - Cause: Payment provider rate limits
   - Impact: Orders can't complete
   - Fix: Queue and retry, contact provider

5. Notification Delivery
   - Symptom: Push notifications delayed
   - Cause: FCM/APNS rate limits
   - Impact: Customers don't know order status
   - Fix: Prioritize critical notifications
```

### Q7: How do you calculate estimated delivery time?

**Answer:**

```
ETA = Prep Time + Partner Travel + Delivery Travel + Buffer

Components:

1. Prep Time:
   - Historical average for restaurant
   - Adjusted by current order volume
   - Item-specific adjustments

2. Partner Travel to Restaurant:
   - If assigned: Actual distance/time
   - If not: Average assignment + travel time
   - Traffic-adjusted using Maps API

3. Delivery Travel:
   - Distance from restaurant to customer
   - Traffic-adjusted
   - Historical accuracy for route

4. Buffer:
   - Base: 5 minutes
   - Peak hours: +10 minutes
   - Bad weather: +15 minutes

```java
public DeliveryEstimate calculateETA(Order order) {
    int prepTime = getPrepTime(order.getRestaurantId(), order.getItems());
    int partnerTravel = getPartnerTravelTime(order);
    int deliveryTravel = getDeliveryTravelTime(order);
    int buffer = getBuffer(order);
    
    int minMinutes = prepTime + deliveryTravel + 5;
    int maxMinutes = prepTime + partnerTravel + deliveryTravel + buffer;
    
    return new DeliveryEstimate(minMinutes, maxMinutes);
}
```

Accuracy improvement:
- ML model trained on historical data
- Factors: time of day, weather, restaurant, route
- Continuously updated with actual delivery times
```

---

## Failure Scenarios

### Q8: What happens if a delivery partner's app crashes mid-delivery?

**Answer:**

```
Scenario: Partner has food, app crashes

Detection:
1. No location updates for 60 seconds
2. WebSocket heartbeat fails
3. System marks partner as "disconnected"

Immediate actions:
1. Customer notified: "Tracking temporarily unavailable"
2. Order continues in "degraded" mode
3. Last known location shown

Recovery paths:

Path A: Partner reconnects within 5 minutes
- App syncs state from server
- Delivery continues normally
- No action needed

Path B: Partner doesn't reconnect
- After 5 minutes, support contacted
- Support calls partner
- If unreachable, reassign delivery

Path C: Partner reports issue
- Partner calls support
- Support can manually update status
- Customer notified of delay

Safety measures:
- Customer can call support
- Restaurant can confirm pickup happened
- GPS history available for disputes
```

### Q9: How do you prevent order fraud?

**Answer:**

```
Fraud vectors and mitigations:

1. Fake Deliveries (partner fraud):
   - GPS verification at pickup/delivery
   - Photo proof of delivery
   - Customer confirmation required
   - ML model flags suspicious patterns

2. Stolen Credit Cards:
   - 3D Secure for new cards
   - Address verification
   - Velocity limits (orders per card per day)
   - Device fingerprinting

3. Promo Code Abuse:
   - One-time use codes
   - Device binding
   - Phone verification
   - ML fraud scoring

4. Fake Restaurants:
   - Verification process
   - Bank account verification
   - Physical address verification
   - Order pattern monitoring

5. Customer Fraud (false complaints):
   - Photo proof of delivery
   - GPS verification
   - History-based trust score
   - Manual review for high-value claims

Detection system:
- Real-time scoring of each order
- ML model trained on historical fraud
- Manual review queue for flagged orders
- Automatic blocks for confirmed fraud
```

### Q10: What if Elasticsearch goes down completely?

**Answer:**

```
Elasticsearch failure is critical but not fatal:

Immediate impact:
- Restaurant search doesn't work
- Menu search doesn't work
- Customers can't discover restaurants

Fallback strategy:

1. PostgreSQL Fallback:
   - Simple queries work
   - No full-text search
   - Basic geo filtering with PostGIS
   - Slower but functional

```java
@CircuitBreaker(name = "elasticsearch", fallbackMethod = "searchFallback")
public List<Restaurant> search(SearchRequest request) {
    return elasticsearchService.search(request);
}

public List<Restaurant> searchFallback(SearchRequest request, Exception e) {
    log.warn("ES unavailable, using PostgreSQL fallback");
    return restaurantRepository.findNearby(
        request.getLat(), 
        request.getLng(),
        request.getRadiusKm()
    );
}
```

2. Cached Results:
   - Serve cached search results
   - Extend TTL during outage
   - Mark results as "may be outdated"

3. Direct Restaurant Access:
   - Customers with saved favorites can order
   - Recent orders accessible
   - Share links still work

Recovery:
- ES cluster self-heals (if partial failure)
- Reindex from PostgreSQL if needed
- Takes 1-2 hours for full reindex
```

---

## Design Evolution Questions

### Q11: What would you build first with 4 weeks?

**Answer:**

```
MVP (4 weeks):

Week 1: Core Data Model
- PostgreSQL schema
- Restaurant and menu CRUD
- Basic user auth

Week 2: Order Flow
- Order placement
- Simple status updates
- Stripe payment integration

Week 3: Restaurant Interface
- Web dashboard for restaurants
- Order acceptance
- Basic menu management

Week 4: Customer App Basics
- Restaurant list (no search)
- Menu viewing
- Order placement
- Order tracking (polling)

What's NOT included:
- Elasticsearch search
- Real-time WebSocket updates
- Delivery partner app
- Recommendations
- Promotions
- Ratings
- Analytics

This MVP:
- Handles ~100 orders/day
- Manual delivery assignment
- Basic functionality only
```

### Q12: How does the design evolve from MVP to production?

**Answer:**

```
Phase 1: MVP (100 orders/day)
├── Single PostgreSQL
├── No caching
├── No search
├── Manual delivery assignment
└── Polling for updates

Phase 2: Growth (1K orders/day)
├── Redis caching
├── Basic Elasticsearch
├── Delivery partner app
├── WebSocket for updates
├── Basic analytics
└── Ratings system

Phase 3: Production (10K orders/day)
├── PostgreSQL replicas
├── Redis cluster
├── Elasticsearch cluster
├── Kafka for events
├── ML-based ETA
├── Fraud detection
└── Full monitoring

Phase 4: Scale (100K orders/day)
├── Multi-region deployment
├── Advanced ML (recommendations)
├── Real-time analytics
├── A/B testing infrastructure
├── Automated operations
└── Cost optimization

Phase 5: Market Leader (1M orders/day)
├── Global expansion
├── Autonomous delivery integration
├── Predictive ordering
├── Kitchen management system
└── Restaurant analytics platform
```

---

## Level-Specific Expectations

### L4 (Entry-Level) Expectations

Should demonstrate:
- Understanding of three-sided marketplace
- Basic API design
- Simple database schema
- Awareness of caching needs

May struggle with:
- Search implementation details
- Event-driven architecture
- Peak hour handling
- Delivery assignment algorithm

### L5 (Mid-Level) Expectations

Should demonstrate:
- Complete end-to-end design
- Elasticsearch for search
- Event-driven with Kafka
- Caching strategy
- Failure scenarios
- Monitoring approach

May struggle with:
- Multi-region deployment
- Advanced ML integration
- Cost optimization at scale
- Complex fraud detection

### L6 (Senior) Expectations

Should demonstrate:
- Multiple architecture approaches
- Deep dive into any component
- Cost analysis
- Design evolution over time
- Edge cases (peak hours, fraud)
- Organizational considerations

---

## Common Interviewer Pushbacks

### "Why not use a single database for everything?"

**Response:**

```
I considered single PostgreSQL, but chose multiple stores because:

1. Different Access Patterns:
   - Search: Full-text + geo (Elasticsearch)
   - Orders: ACID transactions (PostgreSQL)
   - Real-time: Low latency (Redis)

2. Scale Requirements:
   - 290 searches/second at peak
   - PostgreSQL FTS can't handle this
   - ES horizontally scales

3. Latency Requirements:
   - Search must be < 500ms
   - Menu must be < 300ms
   - Redis provides sub-10ms reads

4. Operational Isolation:
   - Search issues don't affect orders
   - Cache failures don't lose data

For a smaller scale (< 1K orders/day):
- Single PostgreSQL would work
- Add Redis for sessions only
- Simpler to operate
```

### "How do you ensure order consistency across services?"

**Response:**

```
We use several patterns for consistency:

1. Outbox Pattern for Events:
   - Order + event in same transaction
   - Events eventually published to Kafka
   - No lost events, no orphan events

2. Idempotent Operations:
   - Every event has unique ID
   - Consumers check before processing
   - Safe to replay events

3. Saga Pattern for Multi-Service:
   - Order creation is single service
   - Payment is separate saga
   - Compensation on failure

4. Read-Your-Writes:
   - After order creation, read from primary
   - Avoid replication lag issues
   - Cache invalidated synchronously

5. Eventual Consistency Acceptance:
   - Search index lags by seconds
   - Analytics lag by minutes
   - Critical paths are consistent
```

### "This seems complex for a food delivery app"

**Response:**

```
You're right to question complexity. Let me justify:

At 500K orders/day:
- 30 orders/second peak
- 290 searches/second peak
- 200K restaurants
- This is DoorDash-scale

Complexity drivers:
1. Three-sided marketplace (customers, restaurants, partners)
2. Real-time requirements (tracking, notifications)
3. Search with geo + text + filters
4. Peak hour handling (5x normal traffic)
5. Payment processing
6. Fraud prevention

For smaller scale (1K orders/day):
- Single PostgreSQL
- Single Redis
- No Elasticsearch (simple filters)
- No Kafka (sync processing)
- Cost: ~$500/month

The architecture I described scales to millions of orders.
I can simplify based on actual requirements.
```

---

## Summary: Key Points to Remember

1. **Three-sided marketplace**: Balance customers, restaurants, partners
2. **Search is critical**: Elasticsearch for discovery
3. **Peak hours**: 5x traffic during meals
4. **Real-time updates**: WebSocket for tracking
5. **Event-driven**: Kafka for decoupling
6. **Caching strategy**: Different TTLs for different data
7. **Outbox pattern**: Transactional event publishing
8. **Graceful degradation**: Prioritize order flow
9. **Cost awareness**: Search and maps are expensive

