# 🎭 Event-Driven Microservices

---

## 0️⃣ Prerequisites

Before diving into event-driven microservices, you should understand:

- **Queue vs Pub/Sub** (Topic 1): Fundamental messaging patterns.
- **Advanced Patterns** (Topic 7): Saga, Event Sourcing, CQRS, Transactional Outbox.
- **Message Delivery** (Topic 2): Delivery guarantees and acknowledgments.
- **Distributed Systems** (Phase 1): CAP theorem, eventual consistency.

**Quick refresher on microservices**: Microservices architecture decomposes an application into small, independent services that communicate over the network. Each service owns its data and can be deployed independently. The challenge is coordinating actions across services without tight coupling.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

In a microservices architecture, services need to communicate. The naive approach causes problems:

```
┌─────────────────────────────────────────────────────────────┐
│              SYNCHRONOUS MICROSERVICES PROBLEMS              │
│                                                              │
│   PROBLEM 1: Tight Coupling                                 │
│   ┌─────────┐     ┌─────────┐     ┌─────────┐              │
│   │  Order  │────►│ Payment │────►│Inventory│              │
│   │ Service │     │ Service │     │ Service │              │
│   └─────────┘     └─────────┘     └─────────┘              │
│                                                              │
│   Order Service KNOWS about Payment and Inventory           │
│   Adding new service = modify Order Service                 │
│   Tight coupling, hard to change                            │
│                                                              │
│   PROBLEM 2: Cascading Failures                             │
│   Inventory Service down → Payment fails → Order fails      │
│   One service failure takes down the whole flow             │
│                                                              │
│   PROBLEM 3: Latency Accumulation                           │
│   Order: 50ms + Payment: 100ms + Inventory: 80ms = 230ms   │
│   Each hop adds latency                                     │
│                                                              │
│   PROBLEM 4: Scalability                                    │
│   Order Service handles 1000 req/s                          │
│   Each request calls 2 other services                       │
│   Payment and Inventory must handle 1000 req/s each         │
│   Scaling is multiplicative                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Event-Driven Solution

```
┌─────────────────────────────────────────────────────────────┐
│              EVENT-DRIVEN ARCHITECTURE                       │
│                                                              │
│   ┌─────────┐                                               │
│   │  Order  │──► "OrderCreated" event                       │
│   │ Service │                                               │
│   └─────────┘         │                                     │
│                       ▼                                     │
│              ┌─────────────────┐                            │
│              │   Event Bus     │                            │
│              │    (Kafka)      │                            │
│              └─────────────────┘                            │
│                 │    │    │                                 │
│                 ▼    ▼    ▼                                 │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐                   │
│   │ Payment │  │Inventory│  │ Email   │                   │
│   │ Service │  │ Service │  │ Service │                   │
│   └─────────┘  └─────────┘  └─────────┘                   │
│                                                              │
│   Order Service doesn't know about other services           │
│   Just publishes event, others subscribe                    │
│   Loose coupling, easy to add new services                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### What Event-Driven Architecture Enables

1. **Loose Coupling**: Services don't know about each other
2. **Resilience**: One service down doesn't block others
3. **Scalability**: Services scale independently
4. **Extensibility**: Add new services without modifying existing ones
5. **Temporal Decoupling**: Producer and consumer don't need to be available simultaneously

---

## 2️⃣ Intuition and Mental Model

### The Newspaper Analogy

```
┌─────────────────────────────────────────────────────────────┐
│              NEWSPAPER ANALOGY                               │
│                                                              │
│   SYNCHRONOUS (Phone calls):                                │
│   Reporter calls editor, editor calls printer, etc.         │
│   Everyone must be available at the same time               │
│   One person unavailable = nothing happens                  │
│                                                              │
│   EVENT-DRIVEN (Newspaper):                                 │
│   Reporter writes article, publishes to newspaper           │
│   Readers subscribe and read when convenient                │
│   Reporter doesn't know who reads                           │
│   New readers can subscribe anytime                         │
│   Reader unavailable? Reads tomorrow's paper                │
│                                                              │
│   ─────────────────────────────────────────────────────────  │
│                                                              │
│   EVENT NOTIFICATION:                                        │
│   "New article published" (just the headline)               │
│   Interested readers fetch the full article                 │
│                                                              │
│   EVENT-CARRIED STATE TRANSFER:                             │
│   "New article published" (includes full article)           │
│   Readers have all the information they need                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Event Types

```
┌─────────────────────────────────────────────────────────────┐
│              EVENT TYPES                                     │
│                                                              │
│   EVENT NOTIFICATION:                                        │
│   "Something happened, here's the ID"                       │
│   {                                                          │
│     "type": "OrderCreated",                                 │
│     "orderId": "O123"                                       │
│   }                                                          │
│   Consumer must fetch details if needed                     │
│   Pros: Small events, simple                                │
│   Cons: Requires callback, coupling to source               │
│                                                              │
│   ─────────────────────────────────────────────────────────  │
│                                                              │
│   EVENT-CARRIED STATE TRANSFER:                             │
│   "Something happened, here's all the data"                 │
│   {                                                          │
│     "type": "OrderCreated",                                 │
│     "orderId": "O123",                                      │
│     "customerId": "C456",                                   │
│     "items": [...],                                         │
│     "total": 150.00                                         │
│   }                                                          │
│   Consumer has all data, no callback needed                 │
│   Pros: Decoupled, consumer autonomy                        │
│   Cons: Larger events, data duplication                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ How It Works Internally

### Event Choreography

In choreography, services react to events without central coordination.

```
┌─────────────────────────────────────────────────────────────┐
│              EVENT CHOREOGRAPHY                              │
│                                                              │
│   Order Flow:                                                │
│                                                              │
│   1. Customer places order                                  │
│      Order Service publishes: "OrderCreated"                │
│                                                              │
│   2. Payment Service sees "OrderCreated"                    │
│      Charges customer                                       │
│      Publishes: "PaymentCompleted"                          │
│                                                              │
│   3. Inventory Service sees "PaymentCompleted"              │
│      Reserves stock                                         │
│      Publishes: "InventoryReserved"                         │
│                                                              │
│   4. Shipping Service sees "InventoryReserved"              │
│      Schedules shipment                                     │
│      Publishes: "ShipmentScheduled"                         │
│                                                              │
│   5. Order Service sees "ShipmentScheduled"                 │
│      Updates order status to "COMPLETED"                    │
│                                                              │
│   ┌─────────┐  OrderCreated   ┌─────────┐                  │
│   │  Order  │ ───────────────►│ Payment │                  │
│   │ Service │                 │ Service │                  │
│   └────▲────┘                 └────┬────┘                  │
│        │                           │ PaymentCompleted       │
│        │ ShipmentScheduled         ▼                       │
│   ┌────┴────┐                 ┌─────────┐                  │
│   │Shipping │◄────────────────│Inventory│                  │
│   │ Service │ InventoryReserved│ Service │                  │
│   └─────────┘                 └─────────┘                  │
│                                                              │
│   Pros: Loose coupling, simple services                     │
│   Cons: Hard to understand flow, distributed logic          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Event Orchestration

In orchestration, a central coordinator directs the flow.

```
┌─────────────────────────────────────────────────────────────┐
│              EVENT ORCHESTRATION                             │
│                                                              │
│   Order Flow:                                                │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              ORDER SAGA ORCHESTRATOR                 │   │
│   │                                                      │   │
│   │   State Machine:                                    │   │
│   │   CREATED → PAYMENT_PENDING → INVENTORY_PENDING     │   │
│   │          → SHIPPING_PENDING → COMPLETED             │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│              │         │         │         │                │
│   Commands:  │         │         │         │                │
│              ▼         ▼         ▼         ▼                │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐         │
│   │ Payment │ │Inventory│ │Shipping │ │  Order  │         │
│   │ Service │ │ Service │ │ Service │ │ Service │         │
│   └─────────┘ └─────────┘ └─────────┘ └─────────┘         │
│              │         │         │         │                │
│   Replies:   │         │         │         │                │
│              ▼         ▼         ▼         ▼                │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              ORDER SAGA ORCHESTRATOR                 │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   Orchestrator sends commands, receives replies             │
│   Knows the full flow, handles failures                     │
│                                                              │
│   Pros: Clear flow, centralized logic, easier debugging     │
│   Cons: Orchestrator is a single point, more coupling       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Eventual Consistency

Event-driven systems are eventually consistent:

```
┌─────────────────────────────────────────────────────────────┐
│              EVENTUAL CONSISTENCY                            │
│                                                              │
│   Time 0: Order created in Order Service                    │
│           Order Service DB: order O123 = CREATED            │
│           Inventory Service DB: (not yet updated)           │
│                                                              │
│   Time 10ms: Event published to Kafka                       │
│                                                              │
│   Time 50ms: Inventory Service receives event               │
│              Inventory Service DB: reserved for O123        │
│                                                              │
│   INCONSISTENCY WINDOW: 0-50ms                              │
│   During this time:                                          │
│   - Order exists in Order Service                           │
│   - Inventory doesn't know about it yet                     │
│                                                              │
│   HANDLING INCONSISTENCY:                                    │
│   1. Accept it (most cases): Brief inconsistency OK         │
│   2. Compensate: If problem detected, fix it                │
│   3. Read from event store: Source of truth                 │
│   4. UI feedback: "Processing..." instead of instant        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Compensating Transactions

When something fails, we need to undo previous steps:

```
┌─────────────────────────────────────────────────────────────┐
│              COMPENSATING TRANSACTIONS                       │
│                                                              │
│   Happy Path:                                                │
│   1. Create Order ✓                                         │
│   2. Charge Payment ✓                                       │
│   3. Reserve Inventory ✓                                    │
│   4. Schedule Shipping ✓                                    │
│   → Order Complete                                          │
│                                                              │
│   Failure at Step 3:                                        │
│   1. Create Order ✓                                         │
│   2. Charge Payment ✓                                       │
│   3. Reserve Inventory ✗ (out of stock!)                   │
│                                                              │
│   Compensation:                                              │
│   - Publish: "InventoryReservationFailed"                   │
│   - Payment Service sees event                              │
│   - Payment Service refunds customer                        │
│   - Order Service sees event                                │
│   - Order Service cancels order                             │
│                                                              │
│   Each action has a compensating action:                    │
│   ┌─────────────────────┬─────────────────────────┐        │
│   │ Action              │ Compensation            │        │
│   ├─────────────────────┼─────────────────────────┤        │
│   │ Create Order        │ Cancel Order            │        │
│   │ Charge Payment      │ Refund Payment          │        │
│   │ Reserve Inventory   │ Release Inventory       │        │
│   │ Schedule Shipping   │ Cancel Shipment         │        │
│   └─────────────────────┴─────────────────────────┘        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Simulation-First Explanation

Let's trace through an order flow in an event-driven system.

### Scenario: E-commerce Order with Choreography

**Services:**
- Order Service
- Payment Service
- Inventory Service
- Notification Service
- Analytics Service

### Happy Path

```
┌─────────────────────────────────────────────────────────────┐
│              HAPPY PATH FLOW                                 │
│                                                              │
│   Time 0ms: Customer clicks "Place Order"                   │
│                                                              │
│   Time 10ms: Order Service                                  │
│   - Creates order (status: PENDING)                         │
│   - Publishes: OrderCreated {orderId: O123, amount: 100}   │
│                                                              │
│   Time 50ms: Events delivered to subscribers                │
│                                                              │
│   Time 60ms: Payment Service                                │
│   - Receives OrderCreated                                   │
│   - Charges customer $100                                   │
│   - Publishes: PaymentCompleted {orderId: O123}            │
│                                                              │
│   Time 60ms: Analytics Service (parallel)                   │
│   - Receives OrderCreated                                   │
│   - Records order for analytics                             │
│   - No event published (end of its flow)                    │
│                                                              │
│   Time 100ms: Inventory Service                             │
│   - Receives PaymentCompleted                               │
│   - Reserves 2 items for O123                               │
│   - Publishes: InventoryReserved {orderId: O123}           │
│                                                              │
│   Time 100ms: Notification Service (parallel)               │
│   - Receives PaymentCompleted                               │
│   - Sends payment confirmation email                        │
│                                                              │
│   Time 150ms: Order Service                                 │
│   - Receives InventoryReserved                              │
│   - Updates order status to CONFIRMED                       │
│   - Publishes: OrderConfirmed {orderId: O123}              │
│                                                              │
│   Time 200ms: Notification Service                          │
│   - Receives OrderConfirmed                                 │
│   - Sends order confirmation email                          │
│                                                              │
│   Result: Order completed, customer notified                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Failure Path with Compensation

```
┌─────────────────────────────────────────────────────────────┐
│              FAILURE PATH FLOW                               │
│                                                              │
│   Time 0ms: Customer clicks "Place Order"                   │
│                                                              │
│   Time 10ms: Order Service                                  │
│   - Creates order (status: PENDING)                         │
│   - Publishes: OrderCreated {orderId: O123}                │
│                                                              │
│   Time 60ms: Payment Service                                │
│   - Receives OrderCreated                                   │
│   - Charges customer $100 ✓                                 │
│   - Publishes: PaymentCompleted {orderId: O123}            │
│                                                              │
│   Time 100ms: Inventory Service                             │
│   - Receives PaymentCompleted                               │
│   - Tries to reserve items                                  │
│   - FAILS: Item out of stock!                               │
│   - Publishes: InventoryReservationFailed {orderId: O123}  │
│                                                              │
│   Time 150ms: Payment Service (compensation)                │
│   - Receives InventoryReservationFailed                     │
│   - Refunds customer $100                                   │
│   - Publishes: PaymentRefunded {orderId: O123}             │
│                                                              │
│   Time 150ms: Order Service (compensation)                  │
│   - Receives InventoryReservationFailed                     │
│   - Updates order status to CANCELLED                       │
│   - Publishes: OrderCancelled {orderId: O123}              │
│                                                              │
│   Time 200ms: Notification Service                          │
│   - Receives OrderCancelled                                 │
│   - Sends cancellation email with refund info               │
│                                                              │
│   Result: Order cancelled, customer refunded and notified   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Uber's Event-Driven Architecture

Uber uses events for:
- Ride lifecycle (requested, matched, started, completed)
- Driver location updates
- Surge pricing signals
- Payment processing

**Pattern**: Choreography for ride events, orchestration for complex flows like disputes.

### Netflix's Event System

Netflix uses events for:
- Viewing events (started, paused, completed)
- Recommendation updates
- A/B test assignments
- Billing events

**Pattern**: Event-carried state transfer for viewing events (includes all context).

### Airbnb's Architecture

Airbnb uses events for:
- Booking lifecycle
- Search indexing
- Pricing updates
- Host/guest messaging

**Pattern**: Mix of choreography and orchestration depending on complexity.

### Shopify's Event Bus

Shopify uses events for:
- Order processing
- Inventory updates
- Webhook deliveries
- App integrations

**Pattern**: Event notification for webhooks, event-carried state for internal events.

---

## 6️⃣ How to Implement or Apply It

### Event Definition

```java
package com.systemdesign.events;

import java.time.Instant;
import java.util.UUID;

/**
 * Base event class.
 */
public abstract class DomainEvent {
    private final String eventId;
    private final String eventType;
    private final Instant occurredAt;
    private final String aggregateId;
    private final int version;
    
    protected DomainEvent(String aggregateId) {
        this.eventId = UUID.randomUUID().toString();
        this.eventType = this.getClass().getSimpleName();
        this.occurredAt = Instant.now();
        this.aggregateId = aggregateId;
        this.version = 1;
    }
    
    // Getters
}

/**
 * Order created event (event-carried state transfer).
 */
public class OrderCreated extends DomainEvent {
    private final String customerId;
    private final List<OrderItem> items;
    private final BigDecimal total;
    private final String currency;
    private final ShippingAddress shippingAddress;
    
    public OrderCreated(String orderId, String customerId, 
                        List<OrderItem> items, BigDecimal total,
                        String currency, ShippingAddress shippingAddress) {
        super(orderId);
        this.customerId = customerId;
        this.items = items;
        this.total = total;
        this.currency = currency;
        this.shippingAddress = shippingAddress;
    }
    
    // Getters
}

/**
 * Payment completed event.
 */
public class PaymentCompleted extends DomainEvent {
    private final String orderId;
    private final String transactionId;
    private final BigDecimal amount;
    private final String currency;
    
    public PaymentCompleted(String orderId, String transactionId,
                           BigDecimal amount, String currency) {
        super(orderId);
        this.orderId = orderId;
        this.transactionId = transactionId;
        this.amount = amount;
        this.currency = currency;
    }
}
```

### Event Publisher with Outbox

```java
package com.systemdesign.events;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * Order service with transactional outbox.
 */
@Service
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        // 1. Create the order
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        order.setItems(request.getItems());
        order.setTotal(calculateTotal(request.getItems()));
        order.setStatus(OrderStatus.PENDING);
        
        Order savedOrder = orderRepository.save(order);
        
        // 2. Create event
        OrderCreated event = new OrderCreated(
            savedOrder.getId(),
            savedOrder.getCustomerId(),
            savedOrder.getItems(),
            savedOrder.getTotal(),
            "USD",
            request.getShippingAddress()
        );
        
        // 3. Save to outbox (same transaction!)
        OutboxEntry outbox = new OutboxEntry();
        outbox.setAggregateType("Order");
        outbox.setAggregateId(savedOrder.getId());
        outbox.setEventType(event.getEventType());
        outbox.setPayload(toJson(event));
        outboxRepository.save(outbox);
        
        return savedOrder;
    }
    
    /**
     * Handle inventory reservation failure - compensate.
     */
    @Transactional
    public void handleInventoryFailed(InventoryReservationFailed event) {
        Order order = orderRepository.findById(event.getOrderId())
            .orElseThrow();
        
        // Update order status
        order.setStatus(OrderStatus.CANCELLED);
        order.setCancellationReason("Inventory unavailable");
        orderRepository.save(order);
        
        // Publish cancellation event
        OrderCancelled cancelledEvent = new OrderCancelled(
            order.getId(),
            "INVENTORY_UNAVAILABLE"
        );
        
        OutboxEntry outbox = new OutboxEntry();
        outbox.setAggregateType("Order");
        outbox.setAggregateId(order.getId());
        outbox.setEventType(cancelledEvent.getEventType());
        outbox.setPayload(toJson(cancelledEvent));
        outboxRepository.save(outbox);
    }
}
```

### Event Consumer

```java
package com.systemdesign.events;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

/**
 * Payment service - consumes order events.
 */
@Service
public class PaymentService {
    
    private final PaymentProcessor paymentProcessor;
    private final PaymentRepository paymentRepository;
    private final OutboxRepository outboxRepository;
    private final InboxRepository inboxRepository;
    
    @KafkaListener(topics = "order-events", groupId = "payment-service")
    @Transactional
    public void handleOrderEvent(String eventJson) {
        DomainEvent event = parseEvent(eventJson);
        
        // Idempotency check
        if (inboxRepository.existsById(event.getEventId())) {
            return;  // Already processed
        }
        
        // Record in inbox
        inboxRepository.save(new InboxEntry(event.getEventId()));
        
        // Route to handler
        if (event instanceof OrderCreated orderCreated) {
            handleOrderCreated(orderCreated);
        } else if (event instanceof InventoryReservationFailed failed) {
            handleInventoryFailed(failed);
        }
    }
    
    private void handleOrderCreated(OrderCreated event) {
        // Process payment
        PaymentResult result = paymentProcessor.charge(
            event.getCustomerId(),
            event.getTotal(),
            event.getCurrency()
        );
        
        // Save payment record
        Payment payment = new Payment();
        payment.setOrderId(event.getAggregateId());
        payment.setTransactionId(result.getTransactionId());
        payment.setAmount(event.getTotal());
        payment.setStatus(PaymentStatus.COMPLETED);
        paymentRepository.save(payment);
        
        // Publish success event
        PaymentCompleted completedEvent = new PaymentCompleted(
            event.getAggregateId(),
            result.getTransactionId(),
            event.getTotal(),
            event.getCurrency()
        );
        
        publishEvent(completedEvent);
    }
    
    private void handleInventoryFailed(InventoryReservationFailed event) {
        // Find the payment
        Payment payment = paymentRepository.findByOrderId(event.getOrderId())
            .orElse(null);
        
        if (payment != null && payment.getStatus() == PaymentStatus.COMPLETED) {
            // Refund
            paymentProcessor.refund(payment.getTransactionId());
            
            payment.setStatus(PaymentStatus.REFUNDED);
            paymentRepository.save(payment);
            
            // Publish refund event
            PaymentRefunded refundEvent = new PaymentRefunded(
                event.getOrderId(),
                payment.getTransactionId(),
                payment.getAmount()
            );
            
            publishEvent(refundEvent);
        }
    }
}
```

### Saga Orchestrator

```java
package com.systemdesign.saga;

import org.springframework.statemachine.StateMachine;
import org.springframework.stereotype.Service;

/**
 * Order saga orchestrator.
 */
@Service
public class OrderSagaOrchestrator {
    
    private final StateMachine<OrderSagaState, OrderSagaEvent> stateMachine;
    private final CommandGateway commandGateway;
    
    public void startSaga(String orderId, OrderCreated event) {
        SagaContext context = new SagaContext(orderId);
        context.setOrderDetails(event);
        
        // Start state machine
        stateMachine.start();
        stateMachine.sendEvent(OrderSagaEvent.ORDER_CREATED);
        
        // Send first command
        ProcessPaymentCommand command = new ProcessPaymentCommand(
            orderId,
            event.getCustomerId(),
            event.getTotal()
        );
        commandGateway.send("payment-commands", command);
    }
    
    @KafkaListener(topics = "saga-replies", groupId = "order-saga")
    public void handleReply(SagaReply reply) {
        switch (reply.getType()) {
            case "PaymentCompleted":
                handlePaymentCompleted(reply);
                break;
            case "PaymentFailed":
                handlePaymentFailed(reply);
                break;
            case "InventoryReserved":
                handleInventoryReserved(reply);
                break;
            case "InventoryFailed":
                handleInventoryFailed(reply);
                break;
        }
    }
    
    private void handlePaymentCompleted(SagaReply reply) {
        stateMachine.sendEvent(OrderSagaEvent.PAYMENT_COMPLETED);
        
        // Next step: Reserve inventory
        ReserveInventoryCommand command = new ReserveInventoryCommand(
            reply.getOrderId(),
            getContext(reply.getOrderId()).getItems()
        );
        commandGateway.send("inventory-commands", command);
    }
    
    private void handleInventoryFailed(SagaReply reply) {
        stateMachine.sendEvent(OrderSagaEvent.INVENTORY_FAILED);
        
        // Compensate: Refund payment
        RefundPaymentCommand command = new RefundPaymentCommand(
            reply.getOrderId()
        );
        commandGateway.send("payment-commands", command);
        
        // Cancel order
        CancelOrderCommand cancelCommand = new CancelOrderCommand(
            reply.getOrderId(),
            "Inventory unavailable"
        );
        commandGateway.send("order-commands", cancelCommand);
    }
}
```

### Application Configuration

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      properties:
        enable.idempotence: true
    consumer:
      group-id: ${spring.application.name}
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.systemdesign.events"

# Event topics
events:
  topics:
    order-events: order-events
    payment-events: payment-events
    inventory-events: inventory-events
    notification-events: notification-events
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

#### 1. Not Handling Eventual Consistency in UI

**Wrong:**
```
User clicks "Place Order"
→ API returns "Order Created"
→ User sees order list
→ Order not there yet! (event not processed)
→ User confused, clicks again
→ Duplicate order!
```

**Right:**
```
User clicks "Place Order"
→ API returns "Order Created" with orderId
→ UI shows "Order O123 is being processed..."
→ Poll or websocket for status updates
→ UI updates when order confirmed
```

#### 2. Missing Idempotency

**Wrong:**
```java
@KafkaListener(topics = "order-events")
public void handleEvent(OrderCreated event) {
    // No idempotency check!
    paymentService.charge(event.getCustomerId(), event.getTotal());
    // If message redelivered, customer charged twice!
}
```

**Right:**
```java
@KafkaListener(topics = "order-events")
@Transactional
public void handleEvent(OrderCreated event) {
    // Check inbox
    if (inboxRepository.existsById(event.getEventId())) {
        return;
    }
    inboxRepository.save(new InboxEntry(event.getEventId()));
    
    paymentService.charge(event.getCustomerId(), event.getTotal());
}
```

#### 3. Event Schema Without Versioning

**Wrong:**
```json
// V1: {"orderId": "O123", "amount": 100}
// V2: {"orderId": "O123", "total": 100}  // Renamed field!

// Old consumers break!
```

**Right:**
```json
// V1: {"orderId": "O123", "amount": 100, "version": 1}
// V2: {"orderId": "O123", "amount": 100, "total": 100, "version": 2}

// Backward compatible - both fields present
```

### Choreography vs Orchestration Trade-offs

| Aspect | Choreography | Orchestration |
|--------|--------------|---------------|
| Coupling | Loose | Tighter to orchestrator |
| Visibility | Hard to trace | Easy to trace |
| Complexity | Distributed | Centralized |
| Single point of failure | No | Orchestrator |
| Adding steps | Modify multiple services | Modify orchestrator |
| Testing | Harder | Easier |

---

## 8️⃣ When NOT to Use This

### When Synchronous is Better

1. **Simple CRUD**: If just saving to database, no need for events
2. **Strong consistency required**: Banking transfers need immediate consistency
3. **Simple request-response**: API that returns computed result
4. **Low latency required**: Event processing adds latency

### Anti-Patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| Event soup | Too many fine-grained events | Aggregate into meaningful events |
| Sync over async | Using events for request-response | Use HTTP for sync needs |
| Missing compensation | No rollback on failure | Design compensation for each step |
| Tight coupling via events | Events contain implementation details | Events should be domain concepts |

---

## 9️⃣ Comparison with Alternatives

### Communication Patterns

| Pattern | Use Case | Consistency | Coupling |
|---------|----------|-------------|----------|
| Synchronous HTTP | Request-response | Strong | Tight |
| Async events | Fire-and-forget | Eventual | Loose |
| Saga (choreography) | Distributed transaction | Eventual | Loose |
| Saga (orchestration) | Distributed transaction | Eventual | Medium |
| Two-phase commit | Distributed transaction | Strong | Tight |

### When to Use Each

```
┌─────────────────────────────────────────────────────────────┐
│              PATTERN SELECTION                               │
│                                                              │
│   Need immediate response?                                   │
│   ├─ Yes: Synchronous HTTP                                  │
│   └─ No: Need to coordinate multiple services?              │
│          ├─ No: Simple async events                         │
│          └─ Yes: Need central visibility?                   │
│                 ├─ Yes: Saga orchestration                  │
│                 └─ No: Saga choreography                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q1: What is event-driven architecture?**

**Answer:**
Event-driven architecture is a design pattern where services communicate by producing and consuming events rather than direct calls.

Key concepts:
- **Event**: Something that happened (OrderCreated, PaymentCompleted)
- **Producer**: Service that publishes events
- **Consumer**: Service that subscribes to events
- **Event bus**: Infrastructure that delivers events (Kafka, RabbitMQ)

Benefits:
- Loose coupling (services don't know about each other)
- Scalability (services scale independently)
- Resilience (one service down doesn't block others)

Trade-off: Eventual consistency instead of immediate consistency.

**Q2: What's the difference between choreography and orchestration?**

**Answer:**
**Choreography:**
- Services react to events independently
- No central coordinator
- Each service knows what to do when it sees an event
- Like a dance where everyone knows their part

**Orchestration:**
- Central coordinator directs the flow
- Sends commands to services, receives replies
- Coordinator knows the full flow
- Like an orchestra with a conductor

Choose choreography for simple flows with loose coupling. Choose orchestration for complex flows where you need visibility and control.

### L5 (Senior) Questions

**Q3: How do you handle failures in event-driven systems?**

**Answer:**
Multiple strategies:

1. **Compensating transactions:**
   - Each action has a compensating action
   - On failure, execute compensations in reverse order
   - Example: Payment fails → release inventory → cancel order

2. **Retry with backoff:**
   - Transient failures: retry with exponential backoff
   - Use dead letter queue for persistent failures

3. **Idempotency:**
   - Make all handlers idempotent
   - Safe to retry without side effects

4. **Saga pattern:**
   - Track saga state
   - On failure, trigger compensation flow

5. **Monitoring and alerting:**
   - Track event processing
   - Alert on failures or stuck sagas

**Q4: How do you ensure exactly-once processing in event-driven systems?**

**Answer:**
True exactly-once delivery is impossible, but we can achieve exactly-once semantics:

1. **Transactional outbox:**
   - Write event to outbox in same transaction as business data
   - Guarantees event published if data committed

2. **Inbox pattern:**
   - Track processed event IDs
   - Skip if already processed

3. **Idempotent handlers:**
   - Processing same event twice has same effect as once
   - Use natural keys, upserts, or idempotency keys

4. **Kafka transactions:**
   - Read-process-write in single transaction
   - Exactly-once within Kafka ecosystem

### L6 (Staff) Questions

**Q5: Design an event-driven order processing system for a large e-commerce platform.**

**Answer:**
Architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                    EVENT-DRIVEN ORDER SYSTEM                 │
│                                                              │
│   ┌─────────┐                                               │
│   │   API   │ ──► Order Service                             │
│   │ Gateway │                │                              │
│   └─────────┘                ▼                              │
│                    ┌─────────────────┐                      │
│                    │  Kafka Cluster  │                      │
│                    │  (Event Bus)    │                      │
│                    └─────────────────┘                      │
│                      │ │ │ │ │ │                            │
│      ┌───────────────┼─┼─┼─┼─┼─┼───────────────┐           │
│      │               │ │ │ │ │ │               │           │
│      ▼               ▼ ▼ ▼ ▼ ▼ ▼               ▼           │
│   ┌──────┐  ┌──────┐ ┌──────┐ ┌──────┐  ┌──────┐          │
│   │Payment│  │Invent│ │Shipp│ │Notif│  │Analyt│          │
│   │Service│  │ory   │ │ing  │ │ication│ │ics   │          │
│   └──────┘  └──────┘ └──────┘ └──────┘  └──────┘          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Key decisions:**

1. **Event types:**
   - OrderCreated (event-carried state)
   - PaymentCompleted, PaymentFailed
   - InventoryReserved, InventoryFailed
   - ShipmentScheduled, ShipmentDelivered

2. **Consistency:**
   - Transactional outbox for reliable publishing
   - Inbox pattern for idempotency
   - Saga for order lifecycle

3. **Failure handling:**
   - Compensating transactions
   - Dead letter queue for failed events
   - Manual intervention for stuck orders

4. **Monitoring:**
   - Event flow tracing (correlation ID)
   - Lag monitoring
   - Business metrics (orders/minute, failure rate)

---

## 1️⃣1️⃣ One Clean Mental Summary

Event-driven microservices communicate through events rather than direct calls, enabling loose coupling, independent scaling, and resilience. Services publish events when something happens (OrderCreated) and subscribe to events they care about. **Choreography** lets services react independently without central coordination—simple but hard to trace. **Orchestration** uses a central coordinator to direct the flow—easier to understand but creates a single point of coordination. **Eventual consistency** is inherent—services may be temporarily inconsistent. Handle failures with **compensating transactions** (undo previous steps) and **idempotent handlers** (safe to retry). Use **transactional outbox** for reliable event publishing and **inbox pattern** for exactly-once processing. Choose event-driven when you need loose coupling and can accept eventual consistency; use synchronous calls when you need immediate consistency or simple request-response.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│           EVENT-DRIVEN MICROSERVICES CHEAT SHEET             │
├─────────────────────────────────────────────────────────────┤
│ EVENT TYPES                                                  │
│   Notification: "Something happened" (ID only)              │
│   State Transfer: "Here's all the data" (full payload)      │
├─────────────────────────────────────────────────────────────┤
│ COORDINATION PATTERNS                                        │
│   Choreography: Services react independently                │
│   Orchestration: Central coordinator directs flow           │
├─────────────────────────────────────────────────────────────┤
│ CONSISTENCY                                                  │
│   Eventual: Brief inconsistency window                      │
│   Compensation: Undo previous steps on failure              │
├─────────────────────────────────────────────────────────────┤
│ RELIABILITY PATTERNS                                         │
│   Outbox: Event + data in same transaction                  │
│   Inbox: Track processed events for idempotency             │
│   Saga: Coordinate multi-step transactions                  │
├─────────────────────────────────────────────────────────────┤
│ FAILURE HANDLING                                             │
│   Retry: Transient failures with backoff                    │
│   Compensate: Undo completed steps                          │
│   DLQ: Park failed events for investigation                 │
├─────────────────────────────────────────────────────────────┤
│ WHEN TO USE                                                  │
│   ✓ Loose coupling needed                                   │
│   ✓ Independent scaling                                     │
│   ✓ Eventual consistency acceptable                         │
│   ✗ Immediate consistency required                          │
│   ✗ Simple request-response                                 │
└─────────────────────────────────────────────────────────────┘
```

