# Common System Design Patterns

## 0️⃣ Prerequisites

Before diving into system design patterns, you should understand:

- **Distributed Systems Basics**: Concepts like replication, partitioning, and consistency (covered in Phases 1 and 5)
- **Data Storage**: SQL vs NoSQL, caching, and message queues (covered in Phases 3, 4, and 6)
- **Problem Approach Framework**: How to structure a system design interview (covered in Topic 1)

Quick refresher: System design patterns are reusable solutions to common architectural problems. They're like design patterns in code, but at the system level. Knowing these patterns helps you quickly identify solutions and discuss tradeoffs intelligently in interviews.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

System design problems often share common challenges:
- How do I handle high read traffic?
- How do I ensure data consistency across services?
- How do I scale writes?
- How do I handle failures gracefully?

Without knowing common patterns, you might:
1. **Reinvent the wheel**: Spend time designing something that already has a well-known solution
2. **Miss tradeoffs**: Not know the standard tradeoffs for common approaches
3. **Lack vocabulary**: Struggle to communicate with the interviewer efficiently
4. **Make poor choices**: Choose approaches that don't fit the problem

### What Breaks Without Pattern Knowledge

**Scenario 1: The Timeline Problem**

A candidate designing a social media feed didn't know about fan-out patterns. They proposed computing the feed on every read by querying all followed users' posts. At scale, this would be impossibly slow. Knowing the fan-out-on-write pattern would have immediately suggested a better approach.

**Scenario 2: The Consistency Problem**

A candidate designing an e-commerce system didn't know about the saga pattern. They proposed using distributed transactions across services, which is complex and doesn't scale. The saga pattern would have been a better fit.

**Scenario 3: The Scaling Problem**

A candidate designing a chat system didn't know about sharding strategies. They proposed a single database, which wouldn't handle the load. Knowing common sharding patterns would have helped them scale the design.

---

## 2️⃣ Intuition and Mental Model

### The Pattern Toolbox

Think of patterns as tools in a toolbox. Each tool is designed for specific problems:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PATTERN TOOLBOX                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  SCALING PATTERNS                                                   │
│  ────────────────                                                   │
│  🔧 Horizontal Scaling      - Add more machines                     │
│  🔧 Vertical Scaling        - Add more resources to one machine     │
│  🔧 Database Sharding       - Split data across databases           │
│  🔧 Read Replicas           - Separate read and write paths         │
│                                                                      │
│  DATA PATTERNS                                                      │
│  ─────────────                                                      │
│  🔧 Fan-out on Write        - Pre-compute at write time             │
│  🔧 Fan-out on Read         - Compute at read time                  │
│  🔧 Write-through Cache     - Update cache on write                 │
│  🔧 Write-behind Cache      - Async cache updates                   │
│  🔧 CQRS                    - Separate read/write models            │
│                                                                      │
│  CONSISTENCY PATTERNS                                               │
│  ────────────────────                                               │
│  🔧 Saga Pattern            - Distributed transactions              │
│  🔧 Event Sourcing          - Store events, derive state            │
│  🔧 Two-Phase Commit        - Atomic distributed operations         │
│  🔧 Eventual Consistency    - Accept temporary inconsistency        │
│                                                                      │
│  RELIABILITY PATTERNS                                               │
│  ────────────────────                                               │
│  🔧 Circuit Breaker         - Fail fast on downstream failures      │
│  🔧 Retry with Backoff      - Handle transient failures             │
│  🔧 Bulkhead                 - Isolate failures                      │
│  🔧 Rate Limiting           - Protect from overload                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### When to Apply Patterns

```
PROBLEM                           PATTERN TO CONSIDER
───────────────────────────────────────────────────────────────────────
High read traffic                 Caching, Read Replicas, CDN
High write traffic                Sharding, Write-behind Cache
Need real-time updates            Fan-out on Write, Pub/Sub
Complex queries on large data     CQRS, Materialized Views
Distributed transactions          Saga, Event Sourcing
Handling failures                 Circuit Breaker, Retry, Bulkhead
Preventing overload               Rate Limiting, Load Shedding
```

---

## 3️⃣ How It Works Internally

Let's explore the most important patterns in detail.

### Pattern 1: Fan-out on Write vs Fan-out on Read

**Problem**: How do you deliver content to many users efficiently?

**Example**: Social media timeline, news feed, notification delivery

#### Fan-out on Write (Push Model)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FAN-OUT ON WRITE                                  │
└─────────────────────────────────────────────────────────────────────┘

  User A posts a tweet
         │
         ▼
  ┌──────────────┐
  │ Tweet Service│
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐     Get followers: [B, C, D, E, F]
  │Fanout Service│────────────────────────────────────┐
  └──────────────┘                                    │
         │                                            │
         │  Push to each follower's timeline cache    │
         │                                            │
    ┌────┴────┬────────┬────────┬────────┐           │
    │         │        │        │        │           │
    ▼         ▼        ▼        ▼        ▼           │
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐        │
│User B│ │User C│ │User D│ │User E│ │User F│        │
│Cache │ │Cache │ │Cache │ │Cache │ │Cache │        │
└──────┘ └──────┘ └──────┘ └──────┘ └──────┘        │
                                                     │
When User B opens app:                               │
  → Read from cache (O(1))                           │
  → Timeline already computed                        │
```

**Pros**:
- Very fast reads (pre-computed)
- Simple read path
- Good for most users

**Cons**:
- Slow writes (must update many caches)
- Celebrity problem (millions of followers = millions of writes)
- Storage overhead (duplicate data)

#### Fan-out on Read (Pull Model)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FAN-OUT ON READ                                   │
└─────────────────────────────────────────────────────────────────────┘

  User A posts a tweet
         │
         ▼
  ┌──────────────┐
  │ Tweet Service│
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Tweet DB    │  ← Just store the tweet (fast write)
  └──────────────┘


When User B opens app:
         │
         ▼
  ┌──────────────┐
  │Timeline Svc  │
  └──────┬───────┘
         │
         │  Get B's following: [A, X, Y, Z]
         │  Query each user's recent tweets
         │  Merge and sort
         │
    ┌────┴────┬────────┬────────┐
    │         │        │        │
    ▼         ▼        ▼        ▼
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│User A│ │User X│ │User Y│ │User Z│
│Tweets│ │Tweets│ │Tweets│ │Tweets│
└──────┘ └──────┘ └──────┘ └──────┘
         │
         ▼
  Merge & Return Timeline
```

**Pros**:
- Fast writes (just store the tweet)
- No celebrity problem
- Less storage

**Cons**:
- Slow reads (must compute on demand)
- Complex read path
- Higher read latency

#### Hybrid Approach (What Twitter Does)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HYBRID APPROACH                                   │
└─────────────────────────────────────────────────────────────────────┘

                    User posts tweet
                           │
                           ▼
                  ┌────────────────┐
                  │ Check follower │
                  │     count      │
                  └───────┬────────┘
                          │
           ┌──────────────┴──────────────┐
           │                             │
    Followers < 10K              Followers >= 10K
    (Regular user)               (Celebrity)
           │                             │
           ▼                             ▼
    Fan-out on Write             Fan-out on Read
    (push to caches)             (store, merge at read)


When reading timeline:
  1. Get pre-computed timeline from cache
  2. Get celebrity tweets (fan-out on read)
  3. Merge and return
```

**When to use which**:
- **Fan-out on Write**: Read-heavy, most users have few followers
- **Fan-out on Read**: Write-heavy, or users have many followers
- **Hybrid**: Large-scale social networks with both regular users and celebrities

---

### Pattern 2: CQRS (Command Query Responsibility Segregation)

**Problem**: How do you optimize for both complex reads and writes?

**Example**: E-commerce with complex product queries, analytics dashboards

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CQRS PATTERN                                      │
└─────────────────────────────────────────────────────────────────────┘

                         ┌─────────────────┐
                         │    API Layer    │
                         └────────┬────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
               Commands                      Queries
               (Write)                       (Read)
                    │                           │
                    ▼                           ▼
           ┌────────────────┐         ┌────────────────┐
           │ Command Handler│         │ Query Handler  │
           └───────┬────────┘         └───────┬────────┘
                   │                          │
                   ▼                          ▼
           ┌────────────────┐         ┌────────────────┐
           │   Write DB     │         │   Read DB      │
           │  (Normalized)  │         │ (Denormalized) │
           │   PostgreSQL   │         │  Elasticsearch │
           └───────┬────────┘         └────────────────┘
                   │                          ▲
                   │                          │
                   └──────────────────────────┘
                        Event/Sync Process
```

**How it works**:

1. **Write Path (Commands)**:
   - Writes go to a normalized database optimized for consistency
   - After write, an event is published
   - Write DB is the source of truth

2. **Read Path (Queries)**:
   - Reads go to a denormalized database optimized for queries
   - Read DB is updated asynchronously from write events
   - Can use different technology (Elasticsearch, Redis, etc.)

3. **Synchronization**:
   - Events sync data from write DB to read DB
   - Eventual consistency between the two

**Example: E-commerce Product Catalog**

```java
// Command: Update product price
public class UpdatePriceCommand {
    private String productId;
    private BigDecimal newPrice;
}

// Command Handler
public class ProductCommandHandler {
    public void handle(UpdatePriceCommand cmd) {
        // 1. Update write DB (PostgreSQL)
        Product product = writeRepo.findById(cmd.getProductId());
        product.setPrice(cmd.getNewPrice());
        writeRepo.save(product);
        
        // 2. Publish event
        eventBus.publish(new PriceUpdatedEvent(
            cmd.getProductId(), 
            cmd.getNewPrice()
        ));
    }
}

// Query: Search products
public class SearchProductsQuery {
    private String keyword;
    private List<String> filters;
}

// Query Handler
public class ProductQueryHandler {
    public List<ProductView> handle(SearchProductsQuery query) {
        // Read from Elasticsearch (optimized for search)
        return elasticsearchRepo.search(
            query.getKeyword(), 
            query.getFilters()
        );
    }
}

// Event Handler: Sync to read DB
public class ProductEventHandler {
    public void handle(PriceUpdatedEvent event) {
        // Update Elasticsearch
        ProductView view = elasticsearchRepo.findById(event.getProductId());
        view.setPrice(event.getNewPrice());
        elasticsearchRepo.save(view);
    }
}
```

**Pros**:
- Optimized read and write paths
- Can scale reads and writes independently
- Different storage technologies for different needs

**Cons**:
- Complexity (two data stores to maintain)
- Eventual consistency (reads may be stale)
- More infrastructure

**When to use**:
- Read and write patterns are very different
- Need complex queries (full-text search, aggregations)
- High read-to-write ratio
- Can tolerate eventual consistency

---

### Pattern 3: Event Sourcing

**Problem**: How do you maintain a complete history of changes and derive current state?

**Example**: Banking transactions, audit logs, collaborative editing

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EVENT SOURCING                                    │
└─────────────────────────────────────────────────────────────────────┘

  TRADITIONAL APPROACH:
  ─────────────────────
  Store current state only:
  
  Account: { id: 123, balance: 500 }
  
  Problem: How did we get to 500? What happened?


  EVENT SOURCING APPROACH:
  ────────────────────────
  Store all events, derive state:
  
  Event Store:
  ┌────────────────────────────────────────────────────────┐
  │ 1. AccountCreated    { id: 123, initial: 0 }          │
  │ 2. MoneyDeposited    { id: 123, amount: 1000 }        │
  │ 3. MoneyWithdrawn    { id: 123, amount: 200 }         │
  │ 4. MoneyWithdrawn    { id: 123, amount: 300 }         │
  └────────────────────────────────────────────────────────┘
  
  Current State = Replay events: 0 + 1000 - 200 - 300 = 500
```

**Architecture**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EVENT SOURCING ARCHITECTURE                       │
└─────────────────────────────────────────────────────────────────────┘

         Command                              Query
            │                                   │
            ▼                                   │
    ┌───────────────┐                          │
    │Command Handler│                          │
    └───────┬───────┘                          │
            │                                   │
            ▼                                   │
    ┌───────────────┐                          │
    │  Event Store  │◀─────────────────────────┤
    │  (Append-only)│                          │
    └───────┬───────┘                          │
            │                                   │
            │ Events                            │
            ▼                                   │
    ┌───────────────┐         ┌───────────────┐│
    │Event Processor│────────▶│  Read Model   ││
    └───────────────┘         │ (Projection)  │◀┘
                              └───────────────┘
```

**Implementation Example**:

```java
// Events
public interface DomainEvent {
    String getAggregateId();
    Instant getTimestamp();
}

public class AccountCreated implements DomainEvent {
    private String accountId;
    private BigDecimal initialBalance;
    private Instant timestamp;
}

public class MoneyDeposited implements DomainEvent {
    private String accountId;
    private BigDecimal amount;
    private Instant timestamp;
}

public class MoneyWithdrawn implements DomainEvent {
    private String accountId;
    private BigDecimal amount;
    private Instant timestamp;
}

// Aggregate (derives state from events)
public class Account {
    private String id;
    private BigDecimal balance;
    private List<DomainEvent> uncommittedEvents = new ArrayList<>();
    
    // Reconstruct from events
    public static Account fromEvents(List<DomainEvent> events) {
        Account account = new Account();
        for (DomainEvent event : events) {
            account.apply(event);
        }
        return account;
    }
    
    private void apply(DomainEvent event) {
        if (event instanceof AccountCreated) {
            this.id = ((AccountCreated) event).getAccountId();
            this.balance = ((AccountCreated) event).getInitialBalance();
        } else if (event instanceof MoneyDeposited) {
            this.balance = this.balance.add(((MoneyDeposited) event).getAmount());
        } else if (event instanceof MoneyWithdrawn) {
            this.balance = this.balance.subtract(((MoneyWithdrawn) event).getAmount());
        }
    }
    
    // Commands produce events
    public void deposit(BigDecimal amount) {
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        MoneyDeposited event = new MoneyDeposited(this.id, amount, Instant.now());
        apply(event);
        uncommittedEvents.add(event);
    }
    
    public void withdraw(BigDecimal amount) {
        if (amount.compareTo(this.balance) > 0) {
            throw new InsufficientFundsException();
        }
        MoneyWithdrawn event = new MoneyWithdrawn(this.id, amount, Instant.now());
        apply(event);
        uncommittedEvents.add(event);
    }
}

// Event Store
public interface EventStore {
    void append(String aggregateId, List<DomainEvent> events);
    List<DomainEvent> getEvents(String aggregateId);
}
```

**Pros**:
- Complete audit trail
- Can reconstruct state at any point in time
- Natural fit for event-driven systems
- Enables temporal queries ("What was the balance on Jan 1?")

**Cons**:
- Complexity
- Event schema evolution is challenging
- Replaying many events can be slow (use snapshots)
- Eventual consistency for read models

**When to use**:
- Need complete audit history
- Domain is naturally event-driven (banking, trading)
- Need temporal queries
- Collaborative systems with conflict resolution

---

### Pattern 4: Saga Pattern

**Problem**: How do you handle distributed transactions across multiple services?

**Example**: E-commerce order processing (inventory, payment, shipping)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    THE PROBLEM                                       │
└─────────────────────────────────────────────────────────────────────┘

  Place Order requires:
  1. Reserve inventory
  2. Process payment
  3. Create shipment
  
  If payment fails, we need to:
  - Release inventory reservation
  - NOT create shipment
  
  Traditional distributed transaction (2PC) doesn't scale well.
  Saga provides an alternative.
```

#### Choreography-based Saga

Each service listens for events and acts accordingly:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CHOREOGRAPHY SAGA                                 │
└─────────────────────────────────────────────────────────────────────┘

  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │  Inventory  │     │   Payment   │     │  Shipping   │
  │   Service   │     │   Service   │     │   Service   │
  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
         │                   │                   │
         │                   │                   │
    ┌────┴───────────────────┴───────────────────┴────┐
    │                  Message Bus                     │
    │                   (Kafka)                        │
    └─────────────────────────────────────────────────┘

  HAPPY PATH:
  ───────────
  1. Order Service → OrderCreated event
  2. Inventory Service listens → reserves stock → InventoryReserved event
  3. Payment Service listens → charges card → PaymentCompleted event
  4. Shipping Service listens → creates shipment → ShipmentCreated event
  5. Order Service listens → marks order complete

  COMPENSATION (Payment fails):
  ─────────────────────────────
  1. Order Service → OrderCreated event
  2. Inventory Service → reserves stock → InventoryReserved event
  3. Payment Service → payment fails → PaymentFailed event
  4. Inventory Service listens → releases stock → InventoryReleased event
  5. Order Service listens → marks order failed
```

#### Orchestration-based Saga

A central orchestrator coordinates the saga:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ORCHESTRATION SAGA                                │
└─────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────┐
                    │ Saga Orchestrator│
                    │  (Order Saga)    │
                    └────────┬─────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
    ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
    │  Inventory  │   │   Payment   │   │  Shipping   │
    │   Service   │   │   Service   │   │   Service   │
    └─────────────┘   └─────────────┘   └─────────────┘

  ORCHESTRATOR LOGIC:
  ────────────────────
  1. Call Inventory.reserve()
     - Success → continue
     - Failure → abort
  
  2. Call Payment.charge()
     - Success → continue
     - Failure → call Inventory.release(), abort
  
  3. Call Shipping.create()
     - Success → complete
     - Failure → call Payment.refund(), call Inventory.release(), abort
```

**Implementation Example (Orchestration)**:

```java
public class OrderSaga {
    
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;
    
    public OrderResult execute(Order order) {
        SagaState state = new SagaState();
        
        try {
            // Step 1: Reserve inventory
            ReservationResult reservation = inventoryService.reserve(
                order.getItems()
            );
            state.setReservationId(reservation.getId());
            
            // Step 2: Process payment
            PaymentResult payment = paymentService.charge(
                order.getCustomerId(),
                order.getTotal()
            );
            state.setPaymentId(payment.getId());
            
            // Step 3: Create shipment
            ShipmentResult shipment = shippingService.create(
                order.getShippingAddress(),
                order.getItems()
            );
            state.setShipmentId(shipment.getId());
            
            return OrderResult.success(order.getId());
            
        } catch (PaymentFailedException e) {
            // Compensate: release inventory
            compensate(state);
            return OrderResult.failed("Payment failed");
            
        } catch (ShippingFailedException e) {
            // Compensate: refund payment, release inventory
            compensate(state);
            return OrderResult.failed("Shipping failed");
        }
    }
    
    private void compensate(SagaState state) {
        if (state.getPaymentId() != null) {
            paymentService.refund(state.getPaymentId());
        }
        if (state.getReservationId() != null) {
            inventoryService.release(state.getReservationId());
        }
    }
}
```

**Choreography vs Orchestration**:

| Aspect | Choreography | Orchestration |
|--------|--------------|---------------|
| Coupling | Loose (event-driven) | Tighter (orchestrator knows all steps) |
| Complexity | Distributed across services | Centralized in orchestrator |
| Visibility | Hard to see full flow | Easy to see full flow |
| Testing | Harder (distributed) | Easier (centralized logic) |
| Single point of failure | No | Yes (orchestrator) |

**When to use**:
- Distributed transactions across services
- Long-running business processes
- Need compensation for failures
- Choreography: Simple flows, loose coupling preferred
- Orchestration: Complex flows, need visibility

---

### Pattern 5: Circuit Breaker

**Problem**: How do you prevent cascading failures when a downstream service is failing?

**Example**: Service A calls Service B. If B is slow/failing, A shouldn't keep trying and exhaust its resources.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CIRCUIT BREAKER STATES                            │
└─────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────┐
                    │     CLOSED      │
                    │  (Normal flow)  │
                    └────────┬────────┘
                             │
                    Failures exceed threshold
                             │
                             ▼
                    ┌─────────────────┐
                    │      OPEN       │
                    │ (Fail fast)     │◀──────────┐
                    └────────┬────────┘           │
                             │                    │
                    After timeout                 │
                             │                    │
                             ▼                    │
                    ┌─────────────────┐           │
                    │   HALF-OPEN     │           │
                    │ (Test request)  │───────────┘
                    └────────┬────────┘    Failure
                             │
                    Success
                             │
                             ▼
                    ┌─────────────────┐
                    │     CLOSED      │
                    │  (Normal flow)  │
                    └─────────────────┘
```

**Implementation**:

```java
public class CircuitBreaker {
    
    private enum State { CLOSED, OPEN, HALF_OPEN }
    
    private State state = State.CLOSED;
    private int failureCount = 0;
    private int successCount = 0;
    private long lastFailureTime = 0;
    
    private final int failureThreshold;      // e.g., 5 failures
    private final long resetTimeout;         // e.g., 30 seconds
    private final int successThreshold;      // e.g., 3 successes to close
    
    public CircuitBreaker(int failureThreshold, long resetTimeout, int successThreshold) {
        this.failureThreshold = failureThreshold;
        this.resetTimeout = resetTimeout;
        this.successThreshold = successThreshold;
    }
    
    public <T> T execute(Supplier<T> action, Supplier<T> fallback) {
        if (state == State.OPEN) {
            if (System.currentTimeMillis() - lastFailureTime > resetTimeout) {
                state = State.HALF_OPEN;
            } else {
                return fallback.get();  // Fail fast
            }
        }
        
        try {
            T result = action.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            return fallback.get();
        }
    }
    
    private synchronized void onSuccess() {
        if (state == State.HALF_OPEN) {
            successCount++;
            if (successCount >= successThreshold) {
                state = State.CLOSED;
                failureCount = 0;
                successCount = 0;
            }
        } else {
            failureCount = 0;
        }
    }
    
    private synchronized void onFailure() {
        failureCount++;
        lastFailureTime = System.currentTimeMillis();
        
        if (failureCount >= failureThreshold) {
            state = State.OPEN;
        }
    }
}

// Usage
CircuitBreaker breaker = new CircuitBreaker(5, 30000, 3);

String result = breaker.execute(
    () -> externalService.call(),           // Primary action
    () -> "Fallback response"               // Fallback
);
```

**Pros**:
- Prevents cascading failures
- Allows failing service time to recover
- Provides fallback behavior

**Cons**:
- Adds complexity
- Need to tune thresholds
- Fallback may not be acceptable for all use cases

**When to use**:
- Calling external services
- Downstream services that may fail
- Want to fail fast rather than wait for timeouts

---

### Pattern 6: Sharding Strategies

**Problem**: How do you distribute data across multiple databases?

**Example**: User data for millions of users

#### Strategy 1: Range-based Sharding

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RANGE-BASED SHARDING                              │
└─────────────────────────────────────────────────────────────────────┘

  User IDs 1-1M        User IDs 1M-2M       User IDs 2M-3M
       │                    │                    │
       ▼                    ▼                    ▼
  ┌─────────┐          ┌─────────┐          ┌─────────┐
  │ Shard 1 │          │ Shard 2 │          │ Shard 3 │
  └─────────┘          └─────────┘          └─────────┘

  Pros:
  - Range queries are efficient (all data in one shard)
  - Easy to understand
  
  Cons:
  - Hot spots (new users all go to latest shard)
  - Uneven distribution over time
```

#### Strategy 2: Hash-based Sharding

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HASH-BASED SHARDING                               │
└─────────────────────────────────────────────────────────────────────┘

  shard_id = hash(user_id) % num_shards

  User 123 → hash(123) % 3 = 1 → Shard 1
  User 456 → hash(456) % 3 = 0 → Shard 0
  User 789 → hash(789) % 3 = 2 → Shard 2

  ┌─────────┐          ┌─────────┐          ┌─────────┐
  │ Shard 0 │          │ Shard 1 │          │ Shard 2 │
  │User 456 │          │User 123 │          │User 789 │
  │  ...    │          │  ...    │          │  ...    │
  └─────────┘          └─────────┘          └─────────┘

  Pros:
  - Even distribution
  - No hot spots
  
  Cons:
  - Range queries require querying all shards
  - Resharding is painful (hash changes)
```

#### Strategy 3: Consistent Hashing

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CONSISTENT HASHING                                │
└─────────────────────────────────────────────────────────────────────┘

  Hash ring with virtual nodes:

              0°
              │
              │    Shard A
        ──────┼──────
       /      │      \
     /        │        \
    │    Shard C    Shard B│
    │         │         │
     \        │        /
       \      │      /
        ──────┼──────
              │
            180°

  Key placement: hash(key) → find next shard clockwise

  Pros:
  - Adding/removing shards only moves ~1/n keys
  - Better than hash-based for dynamic scaling
  
  Cons:
  - More complex implementation
  - Need virtual nodes for even distribution
```

**When to use which**:

| Strategy | Use When |
|----------|----------|
| Range | Need range queries, data has natural ordering |
| Hash | Need even distribution, no range queries |
| Consistent Hash | Dynamic scaling, distributed caching |

---

### Pattern 7: Rate Limiting Strategies

**Problem**: How do you protect your system from being overwhelmed?

#### Strategy 1: Token Bucket

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TOKEN BUCKET                                      │
└─────────────────────────────────────────────────────────────────────┘

  Bucket capacity: 10 tokens
  Refill rate: 2 tokens/second

  ┌─────────────────────────┐
  │ ● ● ● ● ● ● ● ○ ○ ○    │  7 tokens available
  └─────────────────────────┘
  
  Request arrives:
  - If tokens available → consume 1 token, allow request
  - If no tokens → reject request (429 Too Many Requests)
  
  Every second: add 2 tokens (up to max 10)

  Pros:
  - Allows bursts (up to bucket size)
  - Smooth rate limiting
  
  Cons:
  - Slightly more complex than fixed window
```

#### Strategy 2: Sliding Window

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SLIDING WINDOW                                    │
└─────────────────────────────────────────────────────────────────────┘

  Limit: 100 requests per minute
  
  Time: 12:00:30
  Window: 11:59:30 to 12:00:30
  
  ├─────────────────────────────────────────────────────────┤
  11:59:30                                            12:00:30
  
  Count requests in this window.
  If count >= 100, reject.

  Pros:
  - Accurate rate limiting
  - No boundary issues
  
  Cons:
  - More memory (store timestamps)
  - More computation
```

---

## 4️⃣ Simulation: Applying Patterns in an Interview

**Problem**: "Design a notification system"

**Candidate**: "Let me identify which patterns might apply here.

For the core architecture, I see this as an event-driven system. When something happens (new message, order update), we need to notify users across multiple channels (push, email, SMS).

I'll use these patterns:

1. **Event-Driven Architecture**: Services publish events, notification service consumes them
2. **Fan-out on Write**: Pre-compute which users need notifications
3. **Circuit Breaker**: For external providers (FCM, SendGrid, Twilio)
4. **Saga Pattern**: For multi-channel notifications that need coordination

Let me draw this out..."

```
┌─────────────────────────────────────────────────────────────────────┐
│                NOTIFICATION SYSTEM WITH PATTERNS                     │
└─────────────────────────────────────────────────────────────────────┘

  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
  │Order Service│   │Chat Service │   │Social Service│
  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
         │                 │                 │
         │ Events          │                 │
         └─────────────────┼─────────────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │     Kafka       │
                  │ (Event Bus)     │
                  └────────┬────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │  Notification   │
                  │    Service      │
                  └────────┬────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │Push Worker│ │Email Worker│ │SMS Worker│
        │(Circuit   │ │(Circuit   │ │(Circuit  │
        │ Breaker)  │ │ Breaker)  │ │ Breaker) │
        └─────┬─────┘ └─────┬─────┘ └─────┬────┘
              │             │             │
              ▼             ▼             ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │   FCM/   │ │ SendGrid │ │  Twilio  │
        │   APNS   │ │          │ │          │
        └──────────┘ └──────────┘ └──────────┘
```

"The circuit breakers protect us from provider outages. If Twilio is down, we fail fast and don't exhaust our thread pool waiting for timeouts."

---

## 5️⃣ Quick Reference: Pattern Selection Guide

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PATTERN SELECTION GUIDE                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  PROBLEM                          PATTERNS TO CONSIDER              │
│  ───────────────────────────────────────────────────────────────    │
│                                                                      │
│  High read traffic               → Caching, Read Replicas, CDN      │
│  High write traffic              → Sharding, Write-behind Cache     │
│  Social media feed               → Fan-out (Write/Read/Hybrid)      │
│  Complex queries + writes        → CQRS                             │
│  Need audit history              → Event Sourcing                   │
│  Distributed transactions        → Saga Pattern                     │
│  External service failures       → Circuit Breaker                  │
│  Transient failures              → Retry with Backoff               │
│  Resource exhaustion             → Bulkhead, Rate Limiting          │
│  Data distribution               → Sharding (Range/Hash/Consistent) │
│  Real-time updates               → Pub/Sub, WebSockets              │
│  Search functionality            → CQRS with Elasticsearch          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6️⃣ Interview Follow-up Questions WITH Answers

### Q1: "Why would you use fan-out on write vs read?"

**Answer**: "The choice depends on the read/write ratio and follower distribution.

Fan-out on write is better when:
- Reads far exceed writes (social media timelines)
- Most users have a manageable number of followers
- Low read latency is critical

Fan-out on read is better when:
- Users have millions of followers (celebrities)
- Write latency is critical
- Storage is a concern

In practice, Twitter uses a hybrid: fan-out on write for regular users, fan-out on read for celebrities with millions of followers."

### Q2: "When would you NOT use CQRS?"

**Answer**: "CQRS adds complexity, so I'd avoid it when:
- Read and write patterns are similar
- The application is simple CRUD
- Strong consistency is required (CQRS is eventually consistent)
- The team is small and can't maintain two data stores
- The read/write ratio doesn't justify the overhead

CQRS shines when read patterns are very different from write patterns, like complex search queries on data that's written simply."

### Q3: "How do you handle saga failures?"

**Answer**: "Sagas handle failures through compensation. Each step has a compensating action that undoes its effect.

For example, in an order saga:
1. Reserve inventory → Compensate: Release inventory
2. Charge payment → Compensate: Refund payment
3. Create shipment → Compensate: Cancel shipment

If step 2 fails, we run compensation for step 1. If step 3 fails, we run compensation for steps 2 and 1.

The key is making compensations idempotent, since they might be retried. We also need to handle the case where compensation itself fails, typically through manual intervention or a dead letter queue."

### Q4: "What are the tradeoffs of consistent hashing?"

**Answer**: "Consistent hashing has these tradeoffs:

Pros:
- When adding/removing nodes, only ~1/n keys need to move
- Better than simple hash-based sharding for dynamic scaling
- Used by systems like Cassandra, DynamoDB, and memcached

Cons:
- More complex to implement correctly
- Need virtual nodes for even distribution (each physical node maps to multiple points on the ring)
- Still need to handle replication and failover separately
- Hot spots can still occur if keys aren't uniformly distributed"

---

## 🔟 One Clean Mental Summary

System design patterns are reusable solutions to common architectural problems. The most important patterns to know are:

**Data patterns**: Fan-out on write/read for content delivery, CQRS for separating read/write paths, Event Sourcing for audit trails.

**Consistency patterns**: Saga for distributed transactions, eventual consistency for scalability.

**Reliability patterns**: Circuit breaker for handling downstream failures, retry with backoff for transient errors, rate limiting for protection.

**Scaling patterns**: Sharding for data distribution, read replicas for read scaling.

Don't memorize patterns blindly. Understand the problems they solve and their tradeoffs. In interviews, identify the problem first, then select the appropriate pattern. Explain why you chose it and what alternatives you considered.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────┐
│              SYSTEM DESIGN PATTERNS CHEAT SHEET                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  FAN-OUT ON WRITE                                                   │
│  Pre-compute at write time. Fast reads, slow writes.                │
│  Use for: Social feeds with moderate follower counts                │
│                                                                      │
│  FAN-OUT ON READ                                                    │
│  Compute at read time. Fast writes, slower reads.                   │
│  Use for: Celebrity accounts, write-heavy systems                   │
│                                                                      │
│  CQRS                                                               │
│  Separate read and write models/databases.                          │
│  Use for: Complex queries, different read/write patterns            │
│                                                                      │
│  EVENT SOURCING                                                     │
│  Store events, derive state. Complete audit trail.                  │
│  Use for: Banking, audit requirements, temporal queries             │
│                                                                      │
│  SAGA PATTERN                                                       │
│  Distributed transactions via compensation.                         │
│  Use for: Multi-service transactions, order processing              │
│                                                                      │
│  CIRCUIT BREAKER                                                    │
│  Fail fast when downstream is failing.                              │
│  Use for: External service calls, preventing cascading failures     │
│                                                                      │
│  SHARDING                                                           │
│  - Range: Good for range queries, risk of hot spots                 │
│  - Hash: Even distribution, no range queries                        │
│  - Consistent: Dynamic scaling, minimal key movement                │
│                                                                      │
│  RATE LIMITING                                                      │
│  - Token Bucket: Allows bursts, smooth limiting                     │
│  - Sliding Window: Accurate, more memory                            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

