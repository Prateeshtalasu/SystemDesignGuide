# Microservices Communication Patterns

## 0️⃣ Prerequisites

Before diving into this topic, you need to understand:

- **REST vs gRPC**: Synchronous communication mechanisms (Phase 10, Topic 5)
- **Event-Driven Architecture**: Event publishing, consuming, messaging (Phase 6, Topic 14)
- **Message Queues**: Kafka, RabbitMQ basics (Phase 6)
- **Service Discovery**: How services find each other (Phase 10, Topic 4)
- **Distributed Systems**: CAP theorem, eventual consistency (Phase 1)

**Quick refresher**: Microservices need to communicate. Two main approaches: synchronous (request-response, immediate) and asynchronous (events, eventual). This topic covers all patterns and when to use each.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

In a microservices system, services need to work together:

```
Order Service needs to:
  - Check inventory (Inventory Service)
  - Process payment (Payment Service)
  - Create shipment (Shipping Service)
  - Send notification (Notification Service)
```

**The Question**: How should these services communicate?

**Option 1: Synchronous (Request-Response)**
```
Order Service → Inventory Service (wait for response)
Order Service → Payment Service (wait for response)
Order Service → Shipping Service (wait for response)
```

**Option 2: Asynchronous (Events)**
```
Order Service publishes "OrderCreated" event
Inventory Service listens, reserves stock
Payment Service listens, charges customer
Shipping Service listens, creates shipment
```

**Option 3: Hybrid**
```
Order Service → Payment Service (synchronous, needs immediate result)
Order Service publishes "OrderCreated" event (asynchronous, others can process later)
```

### What Breaks Without the Right Pattern

**Using Synchronous for Everything:**
- Tight coupling (services must be available)
- Cascade failures (one service down breaks all)
- Slow (sequential calls)
- Hard to scale independently

**Using Asynchronous for Everything:**
- Eventual consistency (hard to know current state)
- Complex error handling (no immediate feedback)
- Harder to debug (distributed flow)

### Real Examples

**Netflix**: Uses events for most communication (resilient, scalable), synchronous for critical paths.

**Uber**: Hybrid approach - payment is synchronous (needs immediate confirmation), notifications are asynchronous.

**Amazon**: Event-driven for order processing (loose coupling), synchronous for inventory checks.

---

## 2️⃣ Intuition and Mental Model

### The Restaurant Order Analogy

**Synchronous (Request-Response)** is like calling a restaurant:
- You call: "Do you have table for 2?"
- They check: "Yes, 8 PM available"
- You respond: "Reserve it"
- They confirm: "Reserved"
- **You wait for each response before continuing**

**Asynchronous (Events)** is like ordering food delivery:
- You place order online
- Restaurant receives order (event: OrderPlaced)
- Restaurant prepares food (event: FoodPreparing)
- Delivery driver assigned (event: DriverAssigned)
- Food delivered (event: Delivered)
- **Each step happens independently, you get updates**

**Hybrid** is like fine dining:
- You call to reserve (synchronous - need immediate confirmation)
- They send confirmation email (asynchronous - can wait)
- You arrive, order food (synchronous - need to order now)
- They send receipt email (asynchronous - can send later)

### The Key Mental Model

**Synchronous**: "I ask, you answer, I wait" - Immediate, but tightly coupled.

**Asynchronous**: "I announce, you listen when ready" - Loose coupling, but eventual.

**Choose based on:**
- Need immediate response? → Synchronous
- Can accept eventual consistency? → Asynchronous
- Mix of both? → Hybrid

---

## 3️⃣ How It Works Internally

### Pattern 1: Synchronous Communication

**Request-Response (REST/gRPC):**

```
Client Service
  │
  ├─> Send HTTP/gRPC request
  │   "GET /api/users/123"
  │
  ├─> Wait for response (blocking)
  │
  └─> Receive response
      {"id": 123, "name": "John"}
```

**Characteristics:**
- Client waits for response
- Immediate feedback
- Strong consistency (see latest data)
- Tight coupling (both services must be up)

**Use Cases:**
- Need immediate result (payment processing)
- Query operations (get user by ID)
- Operations that need confirmation

### Pattern 2: Asynchronous Communication

**Event-Driven (Messaging):**

```
Producer Service
  │
  ├─> Publish event to message broker
  │   Topic: "order-created"
  │   Payload: {"orderId": "123", ...}
  │
  └─> Continue (doesn't wait)

Message Broker (Kafka/RabbitMQ)
  │
  ├─> Store event
  ├─> Route to subscribers

Consumer Services
  ├─> Inventory Service (subscribed)
  │   └─> Receives event, reserves stock
  │
  ├─> Payment Service (subscribed)
  │   └─> Receives event, charges customer
  │
  └─> Shipping Service (subscribed)
      └─> Receives event, creates shipment
```

**Characteristics:**
- Fire-and-forget (producer doesn't wait)
- Eventual consistency (consumers process when ready)
- Loose coupling (services don't know about each other)
- Resilient (one consumer down doesn't affect others)

**Use Cases:**
- Notifications
- Data replication
- Workflows that can be eventual
- Decoupled processing

### Pattern 3: Request/Reply (Async Request-Response)

**Pattern:**

```
Requestor Service
  │
  ├─> Send request message
  │   Topic: "process-order-request"
  │   Reply-To: "process-order-response"
  │
  └─> Wait for reply message (async)

Processor Service
  │
  ├─> Receive request
  ├─> Process
  └─> Send reply
      Topic: "process-order-response"
      Correlation-ID: (matches request)
```

**Characteristics:**
- Async but request-response semantics
- Decoupled (via message broker)
- More complex (need correlation IDs)

**Use Cases:**
- Long-running operations
- Decoupled request-response

### Pattern 4: Choreography (Event-Driven Coordination)

**How it Works:**

```
Order Service
  │
  ├─> Publishes "OrderCreated" event
  │
Payment Service (subscribed)
  │
  ├─> Receives "OrderCreated"
  ├─> Processes payment
  └─> Publishes "PaymentCompleted" event
      │
Inventory Service (subscribed to "PaymentCompleted")
  │
  ├─> Receives "PaymentCompleted"
  ├─> Reserves stock
  └─> Publishes "InventoryReserved" event
      │
Shipping Service (subscribed to "InventoryReserved")
  │
  └─> Receives "InventoryReserved"
      Creates shipment
```

**Characteristics:**
- No central coordinator
- Services react to events independently
- Loose coupling
- Hard to trace flow
- Hard to handle failures

### Pattern 5: Orchestration (Central Coordinator)

**How it Works:**

```
Order Orchestrator (Central Coordinator)
  │
  ├─> Step 1: Call Inventory Service
  │   "Reserve stock for order 123"
  │   ← Response: "Reserved"
  │
  ├─> Step 2: Call Payment Service
  │   "Charge customer for order 123"
  │   ← Response: "Charged"
  │
  ├─> Step 3: Call Shipping Service
  │   "Create shipment for order 123"
  │   ← Response: "Created"
  │
  └─> Step 4: Update Order Status
      "Order fulfilled"
```

**Characteristics:**
- Central coordinator knows full flow
- Easier to understand and debug
- Single point of coordination
- Tighter coupling (coordinator knows all services)

**Use Cases:**
- Complex workflows
- Need visibility into flow
- Need centralized error handling

---

## 4️⃣ Simulation-First Explanation

### Scenario: Order Processing Flow

**Approach 1: All Synchronous (Anti-Pattern)**

```java
// Order Service
@Service
public class OrderService {
    @Autowired
    private RestTemplate restTemplate;
    
    public OrderResult processOrder(Order order) {
        // Step 1: Check inventory (synchronous)
        InventoryResponse inventory = restTemplate.postForObject(
            "http://inventory-service/api/reserve",
            order.getItems(),
            InventoryResponse.class
        );
        if (!inventory.isAvailable()) {
            return OrderResult.failed("Out of stock");
        }
        
        // Step 2: Process payment (synchronous)
        PaymentResponse payment = restTemplate.postForObject(
            "http://payment-service/api/charge",
            new PaymentRequest(order.getCustomerId(), order.getTotal()),
            PaymentResponse.class
        );
        if (!payment.isSuccess()) {
            // Need to release inventory (compensation)
            restTemplate.post("http://inventory-service/api/release", inventory.getReservationId());
            return OrderResult.failed("Payment failed");
        }
        
        // Step 3: Create shipment (synchronous)
        ShipmentResponse shipment = restTemplate.postForObject(
            "http://shipping-service/api/create",
            new ShipmentRequest(order),
            ShipmentResponse.class
        );
        
        return OrderResult.success(order.getId());
    }
}
```

**Problems:**
- Sequential calls (slow)
- Tight coupling (all services must be up)
- Complex error handling (need compensation)
- Hard to scale independently

**Approach 2: Event Choreography**

```java
// Order Service
@Service
public class OrderService {
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void createOrder(Order order) {
        // Save order
        orderRepository.save(order);
        
        // Publish event
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getItems(),
            order.getTotal()
        );
        kafkaTemplate.send("order-created", event);
        // Doesn't wait, continues
    }
}

// Payment Service (subscribed to "order-created")
@KafkaListener(topics = "order-created")
public void handleOrderCreated(OrderCreatedEvent event) {
    // Process payment
    PaymentResult result = paymentService.charge(
        event.getCustomerId(),
        event.getTotal()
    );
    
    if (result.isSuccess()) {
        // Publish success event
        PaymentCompletedEvent paymentEvent = new PaymentCompletedEvent(
            event.getOrderId(),
            result.getTransactionId()
        );
        kafkaTemplate.send("payment-completed", paymentEvent);
    } else {
        // Publish failure event
        kafkaTemplate.send("payment-failed", new PaymentFailedEvent(event.getOrderId()));
    }
}

// Inventory Service (subscribed to "payment-completed")
@KafkaListener(topics = "payment-completed")
public void handlePaymentCompleted(PaymentCompletedEvent event) {
    // Reserve inventory
    inventoryService.reserve(event.getOrderId());
    kafkaTemplate.send("inventory-reserved", new InventoryReservedEvent(event.getOrderId()));
}

// Shipping Service (subscribed to "inventory-reserved")
@KafkaListener(topics = "inventory-reserved")
public void handleInventoryReserved(InventoryReservedEvent event) {
    // Create shipment
    shippingService.createShipment(event.getOrderId());
}
```

**Benefits:**
- Loose coupling (services don't know about each other)
- Resilient (one service down doesn't break flow)
- Scalable (services scale independently)
- Parallel processing (payment and inventory can happen in parallel if designed)

**Trade-offs:**
- Eventual consistency (hard to know current state)
- Hard to trace flow
- Complex error handling

**Approach 3: Orchestration (Saga Pattern)**

```java
// Order Orchestrator
@Service
public class OrderOrchestrator {
    @Autowired
    private InventoryServiceClient inventoryClient;
    @Autowired
    private PaymentServiceClient paymentClient;
    @Autowired
    private ShippingServiceClient shippingClient;
    
    @Transactional
    public void processOrder(Order order) {
        SagaContext context = new SagaContext(order.getId());
        
        try {
            // Step 1: Reserve inventory
            InventoryResponse inventory = inventoryClient.reserve(order.getItems());
            context.addCompensation(() -> inventoryClient.release(inventory.getReservationId()));
            
            // Step 2: Charge payment
            PaymentResponse payment = paymentClient.charge(order.getCustomerId(), order.getTotal());
            context.addCompensation(() -> paymentClient.refund(payment.getTransactionId()));
            
            // Step 3: Create shipment
            shippingClient.createShipment(order);
            // No compensation needed (last step)
            
            context.markComplete();
        } catch (Exception e) {
            // Compensate in reverse order
            context.compensate();
            throw e;
        }
    }
}
```

**Benefits:**
- Centralized flow (easier to understand)
- Better error handling (compensation built-in)
- Visibility (know exact state)

**Trade-offs:**
- Coordinator knows all services (coupling)
- Coordinator becomes bottleneck

---

## 5️⃣ How Engineers Actually Use This in Production

### Hybrid Approach (Recommended)

**Netflix Pattern:**
- Critical path: Synchronous (payment, authentication)
- Non-critical: Asynchronous (notifications, analytics, logging)

```java
// Order Service
@Service
public class OrderService {
    // Synchronous: Need immediate result
    @Autowired
    private PaymentServiceClient paymentClient;
    
    // Asynchronous: Can be eventual
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public OrderResult createOrder(Order order) {
        // 1. Synchronous: Check inventory (need to know now)
        InventoryResponse inventory = inventoryClient.check(order.getItems());
        if (!inventory.isAvailable()) {
            return OrderResult.failed("Out of stock");
        }
        
        // 2. Synchronous: Process payment (need immediate confirmation)
        PaymentResponse payment = paymentClient.charge(order.getCustomerId(), order.getTotal());
        if (!payment.isSuccess()) {
            return OrderResult.failed("Payment failed");
        }
        
        // 3. Save order
        orderRepository.save(order);
        
        // 4. Asynchronous: Publish events (others can process when ready)
        kafkaTemplate.send("order-created", new OrderCreatedEvent(order));
        
        return OrderResult.success(order.getId());
    }
}

// Notification Service (async consumer)
@KafkaListener(topics = "order-created")
public void sendConfirmationEmail(OrderCreatedEvent event) {
    emailService.sendConfirmation(event.getOrderId(), event.getCustomerId());
}

// Analytics Service (async consumer)
@KafkaListener(topics = "order-created")
public void trackOrder(OrderCreatedEvent event) {
    analyticsService.track("order.created", event);
}
```

### Real-World Implementation

**Spring Boot + REST (Synchronous):**

```java
@FeignClient(name = "payment-service")
public interface PaymentServiceClient {
    @PostMapping("/api/payments")
    PaymentResponse charge(@RequestBody PaymentRequest request);
}
```

**Spring Boot + Kafka (Asynchronous):**

```java
@Service
public class OrderEventPublisher {
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void publishOrderCreated(Order order) {
        OrderCreatedEvent event = new OrderCreatedEvent(order);
        kafkaTemplate.send("order-created", event);
    }
}

@Component
public class OrderEventHandler {
    @KafkaListener(topics = "order-created")
    public void handle(OrderCreatedEvent event) {
        // Process event
    }
}
```

---

## 6️⃣ How to Implement or Apply It

### Decision Framework

**Choose Synchronous when:**
1. Need immediate response
2. Need strong consistency
3. Simple request-response
4. Critical path (payment, authentication)

**Choose Asynchronous when:**
1. Can accept eventual consistency
2. Need loose coupling
3. Notifications, analytics
4. Long-running processes

**Choose Choreography when:**
1. Simple flows
2. Want maximum decoupling
3. Services are independent

**Choose Orchestration when:**
1. Complex workflows
2. Need visibility
3. Need centralized error handling

### Implementation Example

**Hybrid Implementation:**

```java
// Synchronous client
@FeignClient(name = "payment-service")
public interface PaymentClient {
    @PostMapping("/api/payments")
    PaymentResponse charge(@RequestBody PaymentRequest request);
}

// Asynchronous publisher
@Service
public class EventPublisher {
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void publish(String topic, Object event) {
        kafkaTemplate.send(topic, event);
    }
}

// Service using both
@Service
public class OrderService {
    @Autowired
    private PaymentClient paymentClient;  // Synchronous
    @Autowired
    private EventPublisher eventPublisher;  // Asynchronous
    
    public OrderResult createOrder(Order order) {
        // Synchronous: Payment
        PaymentResponse payment = paymentClient.charge(...);
        
        // Asynchronous: Events
        eventPublisher.publish("order-created", new OrderCreatedEvent(order));
        
        return OrderResult.success();
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Pitfalls

**Pitfall 1: Using Synchronous for Everything**

All services call each other synchronously → Cascade failures, tight coupling.

**Solution**: Use asynchronous for non-critical paths.

**Pitfall 2: Using Asynchronous for Everything**

Everything is events → Hard to know current state, complex debugging.

**Solution**: Use synchronous for critical paths that need immediate feedback.

**Pitfall 3: Sync over Async**

Using events but waiting for responses → Defeats purpose of async.

**Solution**: If you need response, use synchronous. Events are fire-and-forget.

### Performance Tradeoffs

| Pattern | Latency | Throughput | Coupling | Consistency |
|---------|---------|------------|----------|-------------|
| Synchronous | High (waits) | Lower | Tight | Strong |
| Asynchronous | Low (fire-forget) | Higher | Loose | Eventual |
| Choreography | Low | High | Very Loose | Eventual |
| Orchestration | Medium | Medium | Medium | Eventual |

---

## 8️⃣ When NOT to Use This

### When Synchronous is Wrong

- Notifications → Use async
- Analytics/logging → Use async
- Long-running processes → Use async

### When Asynchronous is Wrong

- Need immediate confirmation → Use sync
- Strong consistency required → Use sync
- Simple query operations → Use sync

---

## 9️⃣ Comparison with Alternatives

### Communication Patterns Summary

| Pattern | Type | Use Case | Consistency |
|---------|------|----------|-------------|
| REST/gRPC | Synchronous | Request-response | Strong |
| Events | Asynchronous | Fire-and-forget | Eventual |
| Choreography | Asynchronous | Decoupled flows | Eventual |
| Orchestration | Hybrid | Complex workflows | Eventual |
| Request/Reply | Asynchronous | Async request-response | Eventual |

---

## 🔟 Interview Follow-up Questions WITH Answers

### L4 Level Questions

**Q1: When would you use synchronous vs asynchronous communication?**

**Answer**: 
- **Synchronous**: When you need immediate response, strong consistency, or simple request-response (payment processing, authentication, queries).
- **Asynchronous**: When you can accept eventual consistency, need loose coupling, or fire-and-forget operations (notifications, analytics, logging).

**Q2: What's the difference between choreography and orchestration?**

**Answer**:
- **Choreography**: Services react to events independently, no central coordinator. Loose coupling, but hard to trace.
- **Orchestration**: Central coordinator directs the flow. Easier to understand, but coordinator knows all services.

### L5 Level Questions

**Q3: How do you handle failures in asynchronous communication?**

**Answer**:
1. **Idempotent handlers**: Safe to retry
2. **Dead letter queue**: Park failed events for investigation
3. **Compensating transactions**: Undo previous steps
4. **Retry with backoff**: Handle transient failures
5. **Monitoring**: Track event processing, alert on failures

**Q4: When would you use a hybrid approach?**

**Answer**: Use hybrid when you have both critical and non-critical operations:
- **Critical path** (payment, inventory check): Synchronous (need immediate result)
- **Non-critical** (notifications, analytics): Asynchronous (can be eventual)

Example: Order processing - payment is synchronous (need confirmation), notifications are asynchronous (can send later).

### L6 Level Questions

**Q5: Design a communication pattern for an e-commerce order processing system.**

**Answer**:

**Hybrid Approach:**

1. **Synchronous (Critical Path)**:
   - Inventory check (need to know availability)
   - Payment processing (need immediate confirmation)
   - Order creation (need order ID)

2. **Asynchronous (Non-Critical)**:
   - Email notifications
   - Analytics tracking
   - Inventory updates
   - Shipping label generation

**Flow:**
```
User places order
  ↓ (sync)
Order Service: Check inventory → Payment → Create order
  ↓ (async events)
Publish "OrderCreated" event
  ├─> Notification Service: Send confirmation email
  ├─> Analytics Service: Track order metrics
  ├─> Inventory Service: Update inventory (async)
  └─> Shipping Service: Generate shipping label (async)
```

**Reasoning:**
- Payment must be synchronous (need confirmation)
- Notifications can be async (eventual delivery is fine)
- Hybrid balances performance and reliability

---

## 1️⃣1️⃣ One Clean Mental Summary

Microservices communicate via synchronous (request-response, immediate) or asynchronous (events, eventual) patterns. **Synchronous** (REST/gRPC) is for when you need immediate response and strong consistency (payment, queries). **Asynchronous** (events, messaging) is for loose coupling and eventual consistency (notifications, analytics). **Choreography** lets services react to events independently (maximum decoupling). **Orchestration** uses a central coordinator for complex workflows (better visibility). **Hybrid approach** is common: synchronous for critical path, asynchronous for non-critical. Choose based on requirements: need immediate feedback → synchronous, can accept eventual → asynchronous, complex workflow → orchestration, simple flow → choreography. The key trade-off is consistency (strong vs eventual) vs coupling (tight vs loose).




