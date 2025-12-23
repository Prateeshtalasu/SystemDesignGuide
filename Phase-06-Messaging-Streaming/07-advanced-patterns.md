# 🎯 Advanced Messaging Patterns

---

## 0️⃣ Prerequisites

Before diving into advanced patterns, you should understand:

- **Queue vs Pub/Sub** (Topic 1): Fundamental messaging patterns.
- **Message Delivery** (Topic 2): Delivery guarantees and acknowledgments.
- **Kafka Deep Dive** (Topic 5): Topics, partitions, consumer groups.
- **Idempotency** (Phase 1, Topic 13): Making operations safe to retry.
- **Database Transactions** (Phase 3): ACID properties and transaction isolation.

**Quick refresher on the distributed data problem**: In a microservices architecture, each service has its own database. When an operation spans multiple services (like placing an order that affects inventory, payments, and shipping), how do you ensure consistency? You can't use a single database transaction. These patterns solve that problem.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

Imagine an e-commerce order flow:

```
┌─────────────────────────────────────────────────────────────┐
│              THE DISTRIBUTED TRANSACTION PROBLEM             │
│                                                              │
│   Customer places order:                                     │
│   1. Order Service: Create order                            │
│   2. Payment Service: Charge customer                       │
│   3. Inventory Service: Reserve items                       │
│   4. Shipping Service: Schedule delivery                    │
│                                                              │
│   Each service has its own database.                        │
│   What if Payment succeeds but Inventory fails?             │
│                                                              │
│   Without patterns:                                          │
│   - Customer charged                                        │
│   - Items not reserved                                      │
│   - Order in inconsistent state                             │
│   - Manual intervention required                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### What Systems Looked Like Before These Patterns

**Two-Phase Commit (2PC)**:
```
Traditional approach: Distributed transactions
1. Coordinator: "Prepare to commit?"
2. All services: "Ready!"
3. Coordinator: "Commit!"

Problems:
- Blocking (all services wait)
- Single point of failure (coordinator)
- Doesn't scale
- Holding locks across network calls
```

**Direct Service Calls**:
```
Order Service:
  createOrder()
  paymentService.charge()  // What if this fails after order created?
  inventoryService.reserve()  // What if this fails after payment?
  
Problems:
- Partial failures leave inconsistent state
- No automatic recovery
- Tight coupling
```

### What Breaks Without These Patterns

1. **Data Inconsistency**: Order created but payment failed, or payment succeeded but inventory not reserved.

2. **Lost Events**: Service publishes event, crashes before database commit. Event sent but data not saved.

3. **Duplicate Processing**: Network retry causes same event to be processed twice.

4. **No Audit Trail**: Can't trace what happened or replay events.

5. **Tight Coupling**: Services must know about each other, hard to add new services.

### Real Examples of the Problem

**Uber**: A ride involves driver matching, pricing, payment, and notifications. If payment fails after ride completion, how do you handle it?

**Netflix**: Starting a video involves authentication, entitlement check, CDN selection, and playback tracking. These must be coordinated without blocking.

**Amazon**: An order touches inventory, payment, fraud detection, warehouse, shipping. Each can fail independently.

---

## 2️⃣ Intuition and Mental Model

### The Restaurant Kitchen Analogy

Think of these patterns like different ways to run a restaurant kitchen:

```
┌─────────────────────────────────────────────────────────────┐
│              RESTAURANT KITCHEN ANALOGY                      │
│                                                              │
│   TRANSACTIONAL OUTBOX = Order Ticket System                │
│   - Waiter writes order on ticket AND puts in kitchen queue │
│   - Both happen together (same transaction)                 │
│   - Kitchen reads from queue, never misses an order         │
│                                                              │
│   SAGA = Multi-Course Meal                                  │
│   - Each course prepared by different chef                  │
│   - If dessert chef is sick, serve fruit instead (compensate)│
│   - Meal continues, just with adjustments                   │
│                                                              │
│   EVENT SOURCING = Recipe Journal                           │
│   - Don't store "final dish state"                          │
│   - Store every step: "added salt", "stirred 5 min"         │
│   - Can recreate any dish by replaying steps                │
│                                                              │
│   CQRS = Separate Order Taking and Cooking                  │
│   - Waiters optimized for taking orders (writes)            │
│   - Kitchen display optimized for showing orders (reads)    │
│   - Different systems for different purposes                │
│                                                              │
│   CDC = Kitchen Camera                                      │
│   - Camera watches everything that happens                  │
│   - Other systems (inventory, billing) watch the feed       │
│   - No need to explicitly notify them                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ How It Works Internally

### Pattern 1: Transactional Outbox

The Transactional Outbox pattern ensures that database changes and message publishing happen atomically.

```
┌─────────────────────────────────────────────────────────────┐
│              TRANSACTIONAL OUTBOX PATTERN                    │
│                                                              │
│   THE PROBLEM:                                               │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ 1. Save order to database                           │   │
│   │ 2. Publish "OrderCreated" event to Kafka            │   │
│   │                                                      │   │
│   │ What if crash between 1 and 2?                      │   │
│   │ → Order saved, event never published                │   │
│   │ → Downstream services never notified                │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   THE SOLUTION:                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ BEGIN TRANSACTION                                    │   │
│   │   1. INSERT INTO orders (...)                       │   │
│   │   2. INSERT INTO outbox (event_type, payload, ...)  │   │
│   │ COMMIT                                               │   │
│   │                                                      │   │
│   │ Separate process reads outbox, publishes to Kafka   │   │
│   │ Marks outbox entries as published                   │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   FLOW:                                                      │
│   ┌──────────┐    ┌──────────────────────────────────┐      │
│   │ Service  │───►│ Database                          │      │
│   └──────────┘    │ ┌────────────┐  ┌─────────────┐  │      │
│                   │ │   orders   │  │   outbox    │  │      │
│                   │ └────────────┘  └──────┬──────┘  │      │
│                   └────────────────────────┼─────────┘      │
│                                            │                 │
│   ┌──────────────────────────────────────┐│                 │
│   │ Outbox Processor (polls or CDC)      ││                 │
│   │ Reads outbox, publishes to Kafka     │◄┘                 │
│   │ Marks as published                   │                  │
│   └──────────────────┬───────────────────┘                  │
│                      │                                       │
│                      ▼                                       │
│   ┌──────────────────────────────────────┐                  │
│   │              KAFKA                    │                  │
│   │         "order-events" topic         │                  │
│   └──────────────────────────────────────┘                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Outbox Table Schema:**

```sql
CREATE TABLE outbox (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    aggregate_type VARCHAR(255) NOT NULL,  -- "Order", "Payment"
    aggregate_id VARCHAR(255) NOT NULL,    -- "order-123"
    event_type VARCHAR(255) NOT NULL,      -- "OrderCreated"
    payload JSON NOT NULL,                  -- Event data
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    published_at TIMESTAMP NULL,           -- NULL = not yet published
    
    INDEX idx_unpublished (published_at, created_at)
);
```

### Pattern 2: Saga Pattern

The Saga pattern manages distributed transactions through a sequence of local transactions with compensating actions.

```
┌─────────────────────────────────────────────────────────────┐
│                    SAGA PATTERN                              │
│                                                              │
│   Two styles: CHOREOGRAPHY vs ORCHESTRATION                 │
│                                                              │
│   CHOREOGRAPHY (Event-driven):                              │
│   Each service listens to events and decides what to do     │
│                                                              │
│   ┌─────────┐  OrderCreated  ┌─────────┐  PaymentDone       │
│   │  Order  │ ─────────────► │ Payment │ ─────────────►     │
│   │ Service │                │ Service │                    │
│   └─────────┘                └─────────┘                    │
│                                                              │
│       ┌─────────┐  InventoryReserved  ┌─────────┐          │
│   ───►│Inventory│ ──────────────────► │Shipping │          │
│       │ Service │                     │ Service │          │
│       └─────────┘                     └─────────┘          │
│                                                              │
│   ORCHESTRATION (Coordinator-driven):                       │
│   Central orchestrator tells each service what to do        │
│                                                              │
│                    ┌──────────────┐                         │
│                    │ Orchestrator │                         │
│                    │  (Saga)      │                         │
│                    └──────┬───────┘                         │
│              ┌───────────┼───────────┐                      │
│              │           │           │                      │
│              ▼           ▼           ▼                      │
│        ┌─────────┐ ┌─────────┐ ┌─────────┐                 │
│        │ Payment │ │Inventory│ │Shipping │                 │
│        │ Service │ │ Service │ │ Service │                 │
│        └─────────┘ └─────────┘ └─────────┘                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Saga with Compensating Transactions:**

```
┌─────────────────────────────────────────────────────────────┐
│              SAGA COMPENSATION FLOW                          │
│                                                              │
│   Happy Path:                                                │
│   1. Create Order ✓                                         │
│   2. Reserve Inventory ✓                                    │
│   3. Process Payment ✓                                      │
│   4. Schedule Shipping ✓                                    │
│   → Order Complete!                                         │
│                                                              │
│   Failure at Step 3 (Payment Failed):                       │
│   1. Create Order ✓                                         │
│   2. Reserve Inventory ✓                                    │
│   3. Process Payment ✗ (card declined)                      │
│                                                              │
│   Compensation (reverse order):                             │
│   C2. Release Inventory (compensate step 2)                 │
│   C1. Cancel Order (compensate step 1)                      │
│   → Order Cancelled, inventory released                     │
│                                                              │
│   Each step has a compensating action:                      │
│   ┌─────────────────┬─────────────────────────────┐        │
│   │ Action          │ Compensation                │        │
│   ├─────────────────┼─────────────────────────────┤        │
│   │ Create Order    │ Cancel Order                │        │
│   │ Reserve Stock   │ Release Stock               │        │
│   │ Charge Payment  │ Refund Payment              │        │
│   │ Ship Order      │ Cancel Shipment             │        │
│   └─────────────────┴─────────────────────────────┘        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Pattern 3: Event Sourcing

Event Sourcing stores the state of an entity as a sequence of events, not as current state.

```
┌─────────────────────────────────────────────────────────────┐
│                    EVENT SOURCING                            │
│                                                              │
│   TRADITIONAL (State-based):                                │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ orders table                                         │   │
│   │ id: "O1", status: "shipped", total: 150, items: 3   │   │
│   │                                                      │   │
│   │ Only current state. History lost.                   │   │
│   │ "Why is total 150? When did it change?"             │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   EVENT SOURCING:                                            │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ order_events table                                   │   │
│   │ ┌───────────────────────────────────────────────┐   │   │
│   │ │ OrderCreated    {id:"O1", items:[...]}        │   │   │
│   │ │ ItemAdded       {id:"O1", item:"Book", qty:1} │   │   │
│   │ │ ItemRemoved     {id:"O1", item:"Pen"}         │   │   │
│   │ │ PaymentReceived {id:"O1", amount:150}         │   │   │
│   │ │ OrderShipped    {id:"O1", tracking:"XYZ"}     │   │   │
│   │ └───────────────────────────────────────────────┘   │   │
│   │                                                      │   │
│   │ Current state = replay all events                   │   │
│   │ Full history preserved                              │   │
│   │ Can answer "what was state at time T?"              │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   REBUILDING STATE:                                          │
│   events.filter(orderId == "O1")                            │
│         .sortBy(timestamp)                                  │
│         .reduce(applyEvent)                                 │
│         → Current Order State                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Pattern 4: CQRS (Command Query Responsibility Segregation)

CQRS separates read and write models for different optimization.

```
┌─────────────────────────────────────────────────────────────┐
│                    CQRS PATTERN                              │
│                                                              │
│   TRADITIONAL (Single Model):                               │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                    Service                           │   │
│   │              ┌───────────────┐                       │   │
│   │   Writes ───►│  Same Model   │◄─── Reads            │   │
│   │              │  Same Schema  │                       │   │
│   │              └───────────────┘                       │   │
│   │                                                      │   │
│   │   Problem: Reads and writes have different needs    │   │
│   │   - Writes: Validate, enforce rules                 │   │
│   │   - Reads: Fast, denormalized, aggregated           │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   CQRS (Separate Models):                                   │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                                                      │   │
│   │   Commands (Writes)          Queries (Reads)        │   │
│   │        │                          │                 │   │
│   │        ▼                          ▼                 │   │
│   │   ┌─────────┐               ┌─────────┐            │   │
│   │   │ Command │               │  Query  │            │   │
│   │   │ Handler │               │ Handler │            │   │
│   │   └────┬────┘               └────┬────┘            │   │
│   │        │                         │                  │   │
│   │        ▼                         ▼                  │   │
│   │   ┌─────────┐               ┌─────────┐            │   │
│   │   │  Write  │   Events      │  Read   │            │   │
│   │   │  Model  │ ────────────► │  Model  │            │   │
│   │   │(normalized)│            │(denormalized)│        │   │
│   │   └─────────┘               └─────────┘            │   │
│   │                                                      │   │
│   │   Write model: Optimized for consistency            │   │
│   │   Read model: Optimized for queries                 │   │
│   │   Events sync them (eventually consistent)          │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Pattern 5: Change Data Capture (CDC)

CDC captures changes from the database log and publishes them as events.

```
┌─────────────────────────────────────────────────────────────┐
│              CHANGE DATA CAPTURE (CDC)                       │
│                                                              │
│   TRADITIONAL (Application-level events):                   │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Application:                                         │   │
│   │   db.save(order)                                    │   │
│   │   kafka.publish("OrderCreated", order)              │   │
│   │                                                      │   │
│   │ Problems:                                            │   │
│   │   - Must remember to publish event                  │   │
│   │   - Can miss events (bugs, legacy code)             │   │
│   │   - Race conditions between save and publish        │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   CDC (Database-level capture):                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                                                      │   │
│   │   Application ──► Database                          │   │
│   │                      │                              │   │
│   │                      │ Transaction Log              │   │
│   │                      │ (binlog, WAL)                │   │
│   │                      ▼                              │   │
│   │               ┌─────────────┐                       │   │
│   │               │   Debezium  │ (CDC tool)            │   │
│   │               │   Connector │                       │   │
│   │               └──────┬──────┘                       │   │
│   │                      │                              │   │
│   │                      ▼                              │   │
│   │               ┌─────────────┐                       │   │
│   │               │    Kafka    │                       │   │
│   │               │   Topics    │                       │   │
│   │               └─────────────┘                       │   │
│   │                                                      │   │
│   │   Every INSERT, UPDATE, DELETE automatically        │   │
│   │   becomes an event. No application changes needed.  │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Pattern 6: Inbox Pattern

The Inbox pattern ensures idempotent message processing by tracking processed messages.

```
┌─────────────────────────────────────────────────────────────┐
│                    INBOX PATTERN                             │
│                                                              │
│   THE PROBLEM:                                               │
│   Consumer receives message, processes, crashes before ACK  │
│   Message redelivered, processed AGAIN (duplicate!)         │
│                                                              │
│   THE SOLUTION:                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ BEGIN TRANSACTION                                    │   │
│   │   1. Check inbox: Have I seen this message ID?      │   │
│   │      - Yes: Skip processing, ACK message            │   │
│   │      - No: Continue                                 │   │
│   │   2. INSERT INTO inbox (message_id, processed_at)   │   │
│   │   3. Process the message (business logic)           │   │
│   │ COMMIT                                               │   │
│   │                                                      │   │
│   │ If crash after commit: Message in inbox, won't      │   │
│   │ process again on redelivery.                        │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   INBOX TABLE:                                               │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ message_id (PK) │ processed_at │ result            │   │
│   ├─────────────────┼──────────────┼───────────────────┤   │
│   │ msg-123         │ 2024-01-15   │ SUCCESS           │   │
│   │ msg-456         │ 2024-01-15   │ SUCCESS           │   │
│   │ msg-789         │ 2024-01-15   │ FAILED            │   │
│   └─────────────────┴──────────────┴───────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Simulation-First Explanation

Let's trace through a complete order flow using these patterns.

### Scenario: E-commerce Order with Saga

**Setup:**
- Order Service (creates orders)
- Payment Service (charges customers)
- Inventory Service (reserves stock)
- Shipping Service (schedules delivery)

### Happy Path Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    HAPPY PATH                                │
│                                                              │
│   Time 0ms: Customer submits order                          │
│                                                              │
│   Time 10ms: Order Service                                  │
│   BEGIN TRANSACTION                                          │
│     INSERT INTO orders (id, status) VALUES ('O1', 'PENDING')│
│     INSERT INTO outbox (event) VALUES ('OrderCreated')      │
│   COMMIT                                                     │
│                                                              │
│   Time 50ms: Outbox processor publishes to Kafka            │
│   Topic: order-events                                        │
│   Event: {type: "OrderCreated", orderId: "O1", amount: 100} │
│                                                              │
│   Time 100ms: Payment Service receives event                │
│   - Checks inbox: msg-123 not seen                          │
│   BEGIN TRANSACTION                                          │
│     INSERT INTO inbox (msg-123)                             │
│     INSERT INTO payments (orderId: "O1", status: "SUCCESS") │
│     INSERT INTO outbox (event: "PaymentCompleted")          │
│   COMMIT                                                     │
│                                                              │
│   Time 150ms: Inventory Service receives PaymentCompleted   │
│   - Reserves stock                                          │
│   - Publishes "InventoryReserved"                           │
│                                                              │
│   Time 200ms: Shipping Service receives InventoryReserved   │
│   - Schedules shipment                                      │
│   - Publishes "ShipmentScheduled"                           │
│                                                              │
│   Time 250ms: Order Service receives ShipmentScheduled      │
│   - Updates order status to "COMPLETED"                     │
│                                                              │
│   Result: Order completed successfully!                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Failure Path with Compensation

```
┌─────────────────────────────────────────────────────────────┐
│                    FAILURE PATH                              │
│                                                              │
│   Time 0ms: Customer submits order                          │
│                                                              │
│   Time 10ms: Order Service                                  │
│   - Creates order (status: PENDING)                         │
│   - Publishes "OrderCreated"                                │
│                                                              │
│   Time 100ms: Payment Service                               │
│   - Charges customer                                        │
│   - Publishes "PaymentCompleted"                            │
│                                                              │
│   Time 150ms: Inventory Service                             │
│   - Tries to reserve stock                                  │
│   - FAILS: Item out of stock!                               │
│   - Publishes "InventoryReservationFailed"                  │
│                                                              │
│   Time 200ms: Compensation begins                           │
│                                                              │
│   Time 210ms: Payment Service receives failure              │
│   - Refunds customer                                        │
│   - Publishes "PaymentRefunded"                             │
│                                                              │
│   Time 250ms: Order Service receives failure                │
│   - Updates order status to "CANCELLED"                     │
│   - Notifies customer                                       │
│                                                              │
│   Result: Order cancelled, customer refunded                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Uber's Saga Implementation

Uber uses orchestrated sagas for ride booking:

1. **Cadence/Temporal**: Workflow orchestration
2. **Steps**: Match driver → Calculate price → Charge rider → Notify driver
3. **Compensation**: If any step fails, previous steps are compensated

### Netflix's Event Sourcing

Netflix uses event sourcing for:
- User viewing history (events: Started, Paused, Resumed, Completed)
- A/B test assignments (events: Assigned, Converted)
- Allows replay for analytics and debugging

### Airbnb's CQRS

Airbnb uses CQRS for search:
- **Write model**: Normalized listings database
- **Read model**: Elasticsearch for fast search
- **Sync**: Events update search index

### Shopify's Transactional Outbox

Shopify uses outbox pattern for order processing:
- Order and outbox entry in same transaction
- Debezium captures outbox changes
- Events published to Kafka reliably

---

## 6️⃣ How to Implement or Apply It

### Transactional Outbox Implementation

```java
package com.systemdesign.patterns.outbox;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    
    public OrderService(OrderRepository orderRepository, 
                        OutboxRepository outboxRepository) {
        this.orderRepository = orderRepository;
        this.outboxRepository = outboxRepository;
    }
    
    /**
     * Creates order with outbox entry in same transaction.
     * Guarantees both succeed or both fail.
     */
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        // 1. Create the order
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        order.setItems(request.getItems());
        order.setStatus(OrderStatus.PENDING);
        order.setTotal(calculateTotal(request.getItems()));
        
        Order savedOrder = orderRepository.save(order);
        
        // 2. Create outbox entry (same transaction!)
        OutboxEvent event = new OutboxEvent();
        event.setAggregateType("Order");
        event.setAggregateId(savedOrder.getId());
        event.setEventType("OrderCreated");
        event.setPayload(toJson(new OrderCreatedEvent(savedOrder)));
        
        outboxRepository.save(event);
        
        // Both committed together or both rolled back
        return savedOrder;
    }
    
    private String toJson(Object obj) {
        // JSON serialization
        return new ObjectMapper().writeValueAsString(obj);
    }
}

/**
 * Outbox entity.
 */
@Entity
@Table(name = "outbox")
public class OutboxEvent {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String aggregateType;
    private String aggregateId;
    private String eventType;
    
    @Column(columnDefinition = "JSON")
    private String payload;
    
    private LocalDateTime createdAt = LocalDateTime.now();
    private LocalDateTime publishedAt;
    
    // Getters and setters
}

/**
 * Outbox processor - polls and publishes events.
 */
@Service
public class OutboxProcessor {
    
    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    
    @Scheduled(fixedDelay = 1000)  // Every second
    @Transactional
    public void processOutbox() {
        List<OutboxEvent> events = outboxRepository
            .findByPublishedAtIsNullOrderByCreatedAt();
        
        for (OutboxEvent event : events) {
            try {
                // Publish to Kafka
                String topic = event.getAggregateType().toLowerCase() + "-events";
                kafkaTemplate.send(topic, event.getAggregateId(), event.getPayload());
                
                // Mark as published
                event.setPublishedAt(LocalDateTime.now());
                outboxRepository.save(event);
                
            } catch (Exception e) {
                log.error("Failed to publish event: " + event.getId(), e);
                // Will retry on next poll
            }
        }
    }
}
```

### Saga Pattern Implementation (Orchestration)

```java
package com.systemdesign.patterns.saga;

import org.springframework.stereotype.Service;

/**
 * Saga orchestrator for order processing.
 */
@Service
public class OrderSaga {
    
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final ShippingService shippingService;
    private final OrderRepository orderRepository;
    
    /**
     * Executes the order saga with compensation on failure.
     */
    public SagaResult executeOrderSaga(Order order) {
        SagaContext context = new SagaContext(order);
        
        try {
            // Step 1: Reserve inventory
            step1ReserveInventory(context);
            
            // Step 2: Process payment
            step2ProcessPayment(context);
            
            // Step 3: Schedule shipping
            step3ScheduleShipping(context);
            
            // All steps succeeded
            completeOrder(context);
            return SagaResult.success(order);
            
        } catch (SagaStepException e) {
            // Compensate completed steps in reverse order
            compensate(context);
            return SagaResult.failed(e.getMessage());
        }
    }
    
    private void step1ReserveInventory(SagaContext context) {
        try {
            InventoryReservation reservation = inventoryService
                .reserve(context.getOrder().getItems());
            context.setInventoryReservation(reservation);
            context.addCompletedStep(SagaStep.INVENTORY_RESERVED);
        } catch (Exception e) {
            throw new SagaStepException("Inventory reservation failed", e);
        }
    }
    
    private void step2ProcessPayment(SagaContext context) {
        try {
            PaymentResult payment = paymentService
                .charge(context.getOrder().getCustomerId(), 
                        context.getOrder().getTotal());
            context.setPaymentResult(payment);
            context.addCompletedStep(SagaStep.PAYMENT_PROCESSED);
        } catch (Exception e) {
            throw new SagaStepException("Payment processing failed", e);
        }
    }
    
    private void step3ScheduleShipping(SagaContext context) {
        try {
            ShipmentSchedule shipment = shippingService
                .schedule(context.getOrder());
            context.setShipmentSchedule(shipment);
            context.addCompletedStep(SagaStep.SHIPPING_SCHEDULED);
        } catch (Exception e) {
            throw new SagaStepException("Shipping scheduling failed", e);
        }
    }
    
    /**
     * Compensate completed steps in reverse order.
     */
    private void compensate(SagaContext context) {
        List<SagaStep> completedSteps = context.getCompletedSteps();
        Collections.reverse(completedSteps);
        
        for (SagaStep step : completedSteps) {
            try {
                switch (step) {
                    case SHIPPING_SCHEDULED:
                        shippingService.cancel(context.getShipmentSchedule());
                        break;
                    case PAYMENT_PROCESSED:
                        paymentService.refund(context.getPaymentResult());
                        break;
                    case INVENTORY_RESERVED:
                        inventoryService.release(context.getInventoryReservation());
                        break;
                }
            } catch (Exception e) {
                log.error("Compensation failed for step: " + step, e);
                // Log for manual intervention
            }
        }
        
        // Mark order as cancelled
        Order order = context.getOrder();
        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);
    }
    
    private void completeOrder(SagaContext context) {
        Order order = context.getOrder();
        order.setStatus(OrderStatus.COMPLETED);
        orderRepository.save(order);
    }
}

/**
 * Saga context holds state during saga execution.
 */
public class SagaContext {
    private final Order order;
    private final List<SagaStep> completedSteps = new ArrayList<>();
    private InventoryReservation inventoryReservation;
    private PaymentResult paymentResult;
    private ShipmentSchedule shipmentSchedule;
    
    // Constructor, getters, setters
}

enum SagaStep {
    INVENTORY_RESERVED,
    PAYMENT_PROCESSED,
    SHIPPING_SCHEDULED
}
```

### Inbox Pattern Implementation

```java
package com.systemdesign.patterns.inbox;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * Idempotent message processor using inbox pattern.
 */
@Service
public class IdempotentMessageProcessor {
    
    private final InboxRepository inboxRepository;
    private final OrderService orderService;
    
    /**
     * Process message idempotently.
     * Uses inbox table to track processed messages.
     */
    @Transactional
    public void processMessage(String messageId, OrderCreatedEvent event) {
        // 1. Check if already processed
        if (inboxRepository.existsById(messageId)) {
            log.info("Message already processed, skipping: " + messageId);
            return;
        }
        
        // 2. Record in inbox (same transaction as processing)
        InboxEntry entry = new InboxEntry();
        entry.setMessageId(messageId);
        entry.setProcessedAt(LocalDateTime.now());
        entry.setEventType(event.getClass().getSimpleName());
        inboxRepository.save(entry);
        
        // 3. Process the message
        orderService.handleOrderCreated(event);
        
        // If processing fails, transaction rolls back
        // Inbox entry also rolled back
        // Message will be redelivered and reprocessed
    }
}

@Entity
@Table(name = "inbox")
public class InboxEntry {
    @Id
    private String messageId;
    private LocalDateTime processedAt;
    private String eventType;
    
    // Getters and setters
}
```

### Event Sourcing Implementation

```java
package com.systemdesign.patterns.eventsourcing;

/**
 * Event-sourced Order aggregate.
 */
public class Order {
    private String id;
    private OrderStatus status;
    private List<OrderItem> items = new ArrayList<>();
    private double total;
    
    // Events that have been applied
    private final List<OrderEvent> changes = new ArrayList<>();
    
    /**
     * Rebuild order from events.
     */
    public static Order fromEvents(List<OrderEvent> events) {
        Order order = new Order();
        for (OrderEvent event : events) {
            order.apply(event, false);  // Don't record, just replay
        }
        return order;
    }
    
    /**
     * Apply event to update state.
     */
    private void apply(OrderEvent event, boolean isNew) {
        if (event instanceof OrderCreated e) {
            this.id = e.getOrderId();
            this.status = OrderStatus.PENDING;
        } else if (event instanceof ItemAdded e) {
            this.items.add(e.getItem());
            this.total += e.getItem().getPrice();
        } else if (event instanceof ItemRemoved e) {
            this.items.removeIf(i -> i.getId().equals(e.getItemId()));
            this.total -= e.getPrice();
        } else if (event instanceof OrderConfirmed e) {
            this.status = OrderStatus.CONFIRMED;
        } else if (event instanceof OrderShipped e) {
            this.status = OrderStatus.SHIPPED;
        }
        
        if (isNew) {
            changes.add(event);  // Track new events
        }
    }
    
    // Commands that generate events
    
    public void addItem(OrderItem item) {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Cannot modify confirmed order");
        }
        apply(new ItemAdded(this.id, item), true);
    }
    
    public void confirm() {
        if (items.isEmpty()) {
            throw new IllegalStateException("Cannot confirm empty order");
        }
        apply(new OrderConfirmed(this.id), true);
    }
    
    public List<OrderEvent> getUncommittedChanges() {
        return new ArrayList<>(changes);
    }
    
    public void markChangesAsCommitted() {
        changes.clear();
    }
}

/**
 * Event store repository.
 */
@Service
public class EventStore {
    
    private final EventRepository eventRepository;
    
    public void save(String aggregateId, List<OrderEvent> events) {
        for (OrderEvent event : events) {
            EventEntity entity = new EventEntity();
            entity.setAggregateId(aggregateId);
            entity.setEventType(event.getClass().getSimpleName());
            entity.setPayload(toJson(event));
            entity.setTimestamp(LocalDateTime.now());
            eventRepository.save(entity);
        }
    }
    
    public List<OrderEvent> getEvents(String aggregateId) {
        return eventRepository.findByAggregateIdOrderByTimestamp(aggregateId)
            .stream()
            .map(this::toEvent)
            .collect(Collectors.toList());
    }
    
    public Order loadOrder(String orderId) {
        List<OrderEvent> events = getEvents(orderId);
        return Order.fromEvents(events);
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

#### 1. Using Saga Without Idempotency

**Wrong:**
```java
// Saga step without idempotency
void processPayment(Order order) {
    paymentService.charge(order.getCustomerId(), order.getTotal());
    // If retried, customer charged twice!
}
```

**Right:**
```java
// Saga step with idempotency
void processPayment(Order order) {
    String idempotencyKey = "payment-" + order.getId();
    paymentService.charge(order.getCustomerId(), order.getTotal(), idempotencyKey);
    // Retry safe: same key = same result
}
```

#### 2. Event Sourcing Without Snapshots

**Problem:**
```
Order with 10,000 events
Loading order = replay 10,000 events
Very slow!
```

**Solution:**
```
Snapshot every 100 events
Loading = load snapshot + replay 50 events (since snapshot)
Much faster!
```

#### 3. CQRS Without Eventual Consistency Handling

**Problem:**
```
User creates order (write model)
User immediately queries order (read model)
Read model not yet updated!
User sees: "Order not found"
```

**Solution:**
```
- Return created entity from write operation
- Or: Poll read model with retry
- Or: Accept eventual consistency in UI
```

### Pattern Selection Guide

| Pattern | Use When | Don't Use When |
|---------|----------|----------------|
| **Outbox** | Need reliable event publishing | Single database, no events |
| **Saga** | Distributed transactions | Single service, ACID enough |
| **Event Sourcing** | Need audit trail, replay | Simple CRUD, no history needed |
| **CQRS** | Read/write have different needs | Simple queries, low scale |
| **CDC** | Legacy systems, no code changes | Greenfield, can add events |
| **Inbox** | At-least-once delivery | Exactly-once guaranteed |

---

## 8️⃣ When NOT to Use This

### When These Patterns Are Overkill

1. **Simple CRUD applications**: If you have a single database and simple operations, these patterns add unnecessary complexity.

2. **Low-scale systems**: For systems with few users and low throughput, eventual consistency overhead isn't worth it.

3. **Monolithic applications**: If everything is in one service with one database, use database transactions.

4. **Prototypes and MVPs**: Start simple, add patterns when needed.

### Anti-Patterns

1. **Saga for everything**: Not every operation needs a saga. Simple operations can be synchronous.

2. **Event sourcing without clear benefit**: Don't use event sourcing just because it's trendy. Need audit trail or replay? Then consider it.

3. **CQRS for simple queries**: If your reads are simple and your writes are simple, one model is fine.

---

## 9️⃣ Comparison with Alternatives

### Pattern Comparison

| Aspect | Outbox | Saga | Event Sourcing | CQRS |
|--------|--------|------|----------------|------|
| **Problem** | Reliable publishing | Distributed txn | State history | Read/write optimization |
| **Complexity** | Low | Medium | High | Medium |
| **Consistency** | Strong (local) | Eventual | Eventual | Eventual |
| **Audit trail** | No | No | Yes | No |
| **Replay** | No | No | Yes | No |

### When to Combine Patterns

```
┌─────────────────────────────────────────────────────────────┐
│              PATTERN COMBINATIONS                            │
│                                                              │
│   Outbox + Saga:                                            │
│   Each saga step uses outbox for reliable events            │
│                                                              │
│   Event Sourcing + CQRS:                                    │
│   Events are the write model                                │
│   Read model built from events                              │
│                                                              │
│   CDC + Event Sourcing:                                     │
│   CDC captures database changes as events                   │
│   Events stored in event store                              │
│                                                              │
│   Saga + Inbox:                                             │
│   Each saga step uses inbox for idempotency                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q1: What is the Transactional Outbox pattern?**

**Answer:**
The Transactional Outbox pattern ensures that database changes and event publishing happen atomically. Instead of publishing events directly to a message broker (which could fail independently), you write the event to an "outbox" table in the same database transaction as your business data.

A separate process reads the outbox table and publishes events to the message broker. This guarantees that if the business data is saved, the event will eventually be published. If the transaction fails, both the data and the event are rolled back.

**Q2: What is a Saga?**

**Answer:**
A Saga is a pattern for managing distributed transactions across multiple services. Instead of one big transaction, it's a sequence of local transactions where each step publishes an event that triggers the next step.

If any step fails, the saga executes compensating transactions to undo the previous steps. For example, if payment fails after inventory was reserved, the saga releases the inventory.

There are two styles:
- **Choreography**: Services react to events (decentralized)
- **Orchestration**: A central coordinator directs the flow (centralized)

### L5 (Senior) Questions

**Q3: How would you implement exactly-once processing in an event-driven system?**

**Answer:**
Exactly-once processing requires combining several techniques:

1. **Idempotent producers**: Use idempotency keys so retries don't create duplicates.

2. **Inbox pattern**: Track processed message IDs in the database. Before processing, check if the message was already processed.

3. **Transactional processing**: Process the message and record the message ID in the same database transaction.

4. **Outbox for publishing**: Use outbox pattern so publishing and processing are atomic.

The key insight is that true exactly-once delivery is impossible, but we can achieve exactly-once semantics by making processing idempotent.

**Q4: When would you choose choreography vs orchestration for a saga?**

**Answer:**
**Choreography** (event-driven):
- Services are loosely coupled
- Each service owns its logic
- Good for: Simple flows, few steps, independent services
- Risk: Hard to understand full flow, no central visibility

**Orchestration** (coordinator-driven):
- Central orchestrator controls flow
- Easier to understand and monitor
- Good for: Complex flows, many steps, need visibility
- Risk: Orchestrator becomes bottleneck, single point of failure

I'd choose choreography for simple, stable flows where services are truly independent. I'd choose orchestration for complex flows where visibility and control are important, or when the business logic requires coordination.

### L6 (Staff) Questions

**Q5: Design an event-sourced system for a banking application.**

**Answer:**
Banking is a perfect fit for event sourcing due to audit requirements.

**Event types:**
- AccountOpened
- MoneyDeposited
- MoneyWithdrawn
- TransferInitiated
- TransferCompleted
- AccountClosed

**Architecture:**
```
Command → Validate → Store Event → Update Read Model

Event Store:
- Append-only
- Immutable events
- Partitioned by account ID

Read Models:
- Current balance (for queries)
- Transaction history (for statements)
- Fraud detection (for analysis)
```

**Key considerations:**
- Snapshots every N events for performance
- Event versioning for schema evolution
- Idempotency for command handling
- Separate read models for different query patterns

**Q6: How do you handle failures in a saga that spans multiple services?**

**Answer:**
Failure handling in sagas requires careful design:

1. **Compensation design**: Every step needs a compensating action. Design these upfront.

2. **Idempotency**: Both forward actions and compensations must be idempotent. Retries should be safe.

3. **Failure types**:
   - **Transient**: Retry with backoff
   - **Permanent**: Trigger compensation
   - **Unknown**: Retry a few times, then compensate

4. **Compensation failures**: What if compensation fails?
   - Log for manual intervention
   - Retry compensation with backoff
   - Dead letter queue for stuck sagas

5. **Monitoring**: Track saga state, alert on stuck sagas, dashboard for visibility.

6. **Timeout handling**: Set timeouts for each step. If exceeded, treat as failure.

---

## 1️⃣1️⃣ One Clean Mental Summary

Advanced messaging patterns solve the distributed data problem in microservices. **Transactional Outbox** ensures database changes and events are atomic by writing events to an outbox table in the same transaction. **Saga** manages distributed transactions through a sequence of local transactions with compensating actions for rollback. **Event Sourcing** stores state as a sequence of events, enabling replay and audit trails. **CQRS** separates read and write models for independent optimization. **CDC** captures database changes as events without application changes. **Inbox** ensures idempotent message processing by tracking processed message IDs. These patterns trade complexity for reliability in distributed systems. Use them when you need their specific benefits, not by default.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│           ADVANCED PATTERNS CHEAT SHEET                      │
├─────────────────────────────────────────────────────────────┤
│ TRANSACTIONAL OUTBOX                                         │
│   Problem: DB commit + event publish not atomic             │
│   Solution: Write event to outbox table in same txn         │
│   Separate process publishes from outbox                    │
├─────────────────────────────────────────────────────────────┤
│ SAGA                                                         │
│   Problem: Distributed transactions across services         │
│   Solution: Sequence of local txns + compensations          │
│   Choreography: Event-driven, decentralized                 │
│   Orchestration: Coordinator-driven, centralized            │
├─────────────────────────────────────────────────────────────┤
│ EVENT SOURCING                                               │
│   Problem: Need history, audit trail, replay                │
│   Solution: Store events, not state                         │
│   Current state = replay all events                         │
│   Use snapshots for performance                             │
├─────────────────────────────────────────────────────────────┤
│ CQRS                                                         │
│   Problem: Read/write have different needs                  │
│   Solution: Separate models for reads and writes            │
│   Write: Normalized, consistent                             │
│   Read: Denormalized, fast                                  │
├─────────────────────────────────────────────────────────────┤
│ CDC (Change Data Capture)                                    │
│   Problem: Need events from legacy systems                  │
│   Solution: Capture DB log changes as events                │
│   Tools: Debezium, Maxwell, AWS DMS                         │
├─────────────────────────────────────────────────────────────┤
│ INBOX                                                        │
│   Problem: Duplicate message processing                     │
│   Solution: Track processed message IDs in DB               │
│   Check before process, record in same txn                  │
├─────────────────────────────────────────────────────────────┤
│ COMMON COMBINATIONS                                          │
│   Outbox + Saga: Reliable saga events                       │
│   Event Sourcing + CQRS: Events as write model              │
│   Saga + Inbox: Idempotent saga steps                       │
└─────────────────────────────────────────────────────────────┘
```

