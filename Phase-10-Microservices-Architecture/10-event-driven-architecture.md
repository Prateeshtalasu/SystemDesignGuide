# Event-Driven Architecture

## 0️⃣ Prerequisites

Before diving into Event-Driven Architecture (EDA), you should understand:

- **Microservices fundamentals**: What microservices are, why we use them, and how they communicate (covered in topics 1-6)
- **Message brokers**: Basic understanding of message queues and pub/sub systems like Kafka, RabbitMQ, or AWS SNS/SQS (covered in Phase 6)
- **Database transactions**: ACID properties and why distributed transactions are challenging (covered in topic 7)
- **Domain-Driven Design**: Bounded contexts and domain events (covered in topic 3)

If you're not familiar with these, review them first. Event-Driven Architecture builds on these concepts.

---

## 1️⃣ What problem does this exist to solve?

### The Pain Points

Traditional request-response architectures create several problems:

**Problem 1: Tight Coupling**
- Service A needs to know about Service B, C, and D
- If Service B changes its API, Service A breaks
- Services must be available simultaneously for a workflow to complete
- One service failure cascades to others

**Problem 2: Synchronous Bottlenecks**
- User request waits for 5 services to respond sequentially
- Each service adds latency (50ms + 100ms + 200ms = 350ms total)
- Slowest service determines overall response time
- Services can't work independently

**Problem 3: Data Consistency Across Services**
- Order service needs to update inventory, payment, and shipping
- If payment succeeds but inventory update fails, what happens?
- Distributed transactions (2PC) are slow and complex
- Services need each other's data but can't share databases

**Problem 4: Scalability Challenges**
- All services must scale together
- Can't scale "send email" service independently from "process payment"
- Traffic spikes in one area affect everything

**Real-World Example: E-commerce Order Processing**

In a traditional synchronous system:
```
User clicks "Place Order"
  → Order Service (creates order)
    → Payment Service (processes payment) - waits 2 seconds
      → Inventory Service (reserves items) - waits 1 second
        → Shipping Service (creates shipment) - waits 1 second
          → Email Service (sends confirmation) - waits 500ms
            → Response to user (total: 4.5 seconds)
```

If Email Service is down, the entire order fails. If Payment Service is slow, everything waits.

**What breaks without Event-Driven Architecture:**
- Services become tightly coupled
- System becomes fragile (one failure breaks everything)
- Can't scale components independently
- Hard to add new features (must modify multiple services)
- Difficult to replay or audit what happened

---

## 2️⃣ Intuition and mental model

### The Restaurant Analogy

Think of Event-Driven Architecture like a modern restaurant kitchen:

**Traditional (Synchronous) = Fast Food Drive-Through**
- Customer orders at window 1
- Window 1 calls window 2: "Make burger!"
- Window 2 calls window 3: "Add fries!"
- Window 3 calls window 4: "Add drink!"
- Each step waits for the previous one
- If window 3 breaks, everything stops

**Event-Driven = Fine Dining Kitchen**
- Customer places order (event: "OrderPlaced")
- Order ticket goes to the kitchen board (event bus)
- Chef sees "OrderPlaced" → starts cooking (subscribes to event)
- Fry cook sees "OrderPlaced" → starts fries (also subscribes)
- Bartender sees "OrderPlaced" → prepares drink (also subscribes)
- Each person works independently
- When done, they post their own events: "BurgerReady", "FriesReady", "DrinkReady"
- Waiter subscribes to all "Ready" events → assembles order when all complete

**Key Differences:**
- In event-driven: Services don't call each other directly
- Services publish events: "Something happened"
- Other services subscribe: "I care about that, I'll react"
- Services work in parallel, not sequentially
- Services don't need to know who else is listening

**The Event Bus = The Kitchen Board**
- All events are posted publicly
- Anyone can read and react
- New services can join without modifying existing ones
- Events are immutable facts: "Order #123 was placed at 2:30 PM"

---

## 3️⃣ How it works internally

### Core Components

**1. Event Producer**
- Service that does something (creates order, processes payment)
- Publishes an event when something happens
- Doesn't know who will consume the event

**2. Event Bus/Broker**
- Central message broker (Kafka, RabbitMQ, AWS EventBridge)
- Receives events from producers
- Routes events to subscribers
- Stores events (for replay, audit, late subscribers)

**3. Event Consumer**
- Service that subscribes to events
- Reacts when relevant events arrive
- Can publish new events as a result

**4. Event Store (optional, for Event Sourcing)**
- Immutable log of all events
- Source of truth for system state
- Can replay events to rebuild state

### Event Lifecycle

**Step 1: Event Creation**
```
Order Service receives "Place Order" request
  → Validates order
  → Creates order in database
  → Publishes event: {
      "eventType": "OrderPlaced",
      "orderId": "12345",
      "userId": "user-789",
      "items": [...],
      "timestamp": "2024-01-15T10:30:00Z"
    }
```

**Step 2: Event Publishing**
```
Order Service → Event Bus
  - Serializes event (JSON, Avro, Protobuf)
  - Sends to topic/queue: "order-events"
  - Gets acknowledgment from broker
  - Continues processing (doesn't wait for consumers)
```

**Step 3: Event Routing**
```
Event Bus receives event
  - Stores in topic partition
  - Checks subscriptions: "Who wants OrderPlaced events?"
  - Routes to: Payment Service, Inventory Service, Analytics Service
  - Each gets a copy (pub/sub) or competes for it (queue)
```

**Step 4: Event Consumption**
```
Payment Service receives event
  - Deserializes event
  - Validates event structure
  - Processes payment
  - Publishes new event: "PaymentProcessed"
  - Acknowledges event (removes from queue)
```

**Step 5: Event Propagation**
```
PaymentProcessed event triggers:
  - Shipping Service: "Start preparing shipment"
  - Email Service: "Send payment confirmation"
  - Analytics Service: "Record payment metric"
```

### Event Types

**1. Event Notification (Lightweight)**
```json
{
  "eventType": "OrderPlaced",
  "orderId": "12345",
  "timestamp": "2024-01-15T10:30:00Z"
}
```
- Just tells you something happened
- Consumer must fetch full data if needed
- Small payload, fast

**2. Event-Carried State Transfer (Heavy)**
```json
{
  "eventType": "OrderPlaced",
  "orderId": "12345",
  "order": {
    "id": "12345",
    "userId": "user-789",
    "items": [
      {"productId": "prod-1", "quantity": 2, "price": 29.99}
    ],
    "total": 59.98,
    "shippingAddress": {...}
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```
- Contains all data consumer might need
- Consumer doesn't need to call back to source
- Larger payload, but self-contained

**3. Domain Event (DDD Style)**
```json
{
  "eventType": "OrderPlaced",
  "aggregateId": "order-12345",
  "aggregateType": "Order",
  "eventVersion": 1,
  "payload": {...},
  "metadata": {
    "correlationId": "req-abc-123",
    "causationId": "evt-xyz-789"
  }
}
```
- Rich metadata for tracing
- Versioned for evolution
- Links events in a chain

---

## 4️⃣ Simulation-first explanation

### Simplest Possible System

Let's build the simplest event-driven system: **One producer, one consumer, one message broker.**

**Setup:**
- Order Service (producer)
- Email Service (consumer)
- Kafka (event broker)
- One topic: "order-events"

**Step-by-Step Flow:**

**1. User places order**
```
User → Order Service: POST /orders
  Body: {"userId": "user-1", "items": [...]}
```

**2. Order Service processes order**
```java
// Order Service
@RestController
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private EventPublisher eventPublisher;
    
    @PostMapping("/orders")
    public OrderResponse placeOrder(@RequestBody OrderRequest request) {
        // 1. Create order in database
        Order order = orderService.createOrder(request);
        
        // 2. Publish event (async, non-blocking)
        OrderPlacedEvent event = new OrderPlacedEvent(
            order.getId(),
            order.getUserId(),
            order.getItems(),
            Instant.now()
        );
        eventPublisher.publish("order-events", event);
        
        // 3. Return immediately (don't wait for email)
        return new OrderResponse(order.getId(), "PENDING");
    }
}
```

**3. Event Publisher sends to Kafka**
```java
// Event Publisher
@Component
public class KafkaEventPublisher implements EventPublisher {
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    public void publish(String topic, Object event) {
        String json = objectMapper.writeValueAsString(event);
        kafkaTemplate.send(topic, json);
        // Returns immediately, doesn't wait
    }
}
```

**What happens in Kafka:**
```
Topic: order-events
Partition 0: [Event1, Event2, Event3, ...]
              ↑
              Current offset: 2
```

**4. Email Service consumes event**
```java
// Email Service
@Component
public class OrderEventListener {
    
    @KafkaListener(topics = "order-events")
    public void handleOrderPlaced(String eventJson) {
        // 1. Deserialize
        OrderPlacedEvent event = objectMapper.readValue(
            eventJson, 
            OrderPlacedEvent.class
        );
        
        // 2. Process
        sendOrderConfirmationEmail(
            event.getUserId(),
            event.getOrderId()
        );
        
        // 3. Acknowledge (Kafka moves offset forward)
    }
    
    private void sendOrderConfirmationEmail(String userId, String orderId) {
        // Send email via SMTP/SES
        emailService.send(userId, "Your order #" + orderId + " was placed!");
    }
}
```

**Timeline:**
```
T=0ms:   User clicks "Place Order"
T=50ms:  Order Service saves to DB
T=55ms:  Order Service publishes event to Kafka
T=56ms:  Order Service returns response to user ✅
T=60ms:  Kafka stores event
T=100ms: Email Service receives event
T=150ms: Email Service sends email
T=200ms: Email Service acknowledges (offset++)
```

**Key Insight:** User got response at 56ms, email sent at 150ms. They're decoupled!

### Adding More Consumers

Now add Inventory Service without changing Order Service:

**Email Service (existing):**
```java
@KafkaListener(topics = "order-events")
public void handleOrderPlaced(OrderPlacedEvent event) {
    sendEmail(event);
}
```

**Inventory Service (new):**
```java
@KafkaListener(topics = "order-events")
public void handleOrderPlaced(OrderPlacedEvent event) {
    reserveInventory(event.getItems());
}
```

**What happens:**
```
Order Service publishes 1 event
  ↓
Kafka duplicates it
  ↓
  ├─→ Email Service (gets copy)
  └─→ Inventory Service (gets copy)
```

Both services process in parallel. Order Service doesn't know about Inventory Service.

---

## 5️⃣ How engineers actually use this in production

### Real-World Implementations

**Netflix: Event-Driven Microservices**
- Uses Kafka for event streaming
- Every user action (play, pause, search) becomes an event
- Analytics, recommendations, and billing all consume events
- Can replay events to rebuild state
- Handles millions of events per second

**Uber: Event-Driven Architecture**
- Trip events: "TripStarted", "TripCompleted", "PaymentProcessed"
- Multiple services react: Driver app, Rider app, Billing, Analytics
- Uses Apache Kafka for event streaming
- Event sourcing for trip history (can replay to see exact trip)

**Amazon: Event-Driven Order Processing**
- Order events trigger: Inventory, Shipping, Payment, Recommendations
- Uses AWS EventBridge and SQS
- Events flow through multiple services asynchronously
- Can handle Black Friday traffic spikes

**Spotify: Event-Driven Music Platform**
- Play events, skip events, like events
- Real-time recommendations based on events
- Analytics and reporting from event streams
- Uses Google Pub/Sub and Kafka

### Production Workflows

**1. Event Schema Evolution**
```java
// Version 1
{
  "eventType": "OrderPlaced",
  "orderId": "123",
  "userId": "user-1"
}

// Version 2 (adds new field)
{
  "eventType": "OrderPlaced",
  "orderId": "123",
  "userId": "user-1",
  "promoCode": "SAVE10"  // NEW
}
```

**Strategy:**
- Use schema registry (Confluent Schema Registry, AWS Glue)
- Consumers handle both versions
- Backward compatible changes only
- Deprecate old versions gradually

**2. Event Ordering**
- Kafka: Events in same partition are ordered
- Partition by orderId → all events for order #123 stay in order
- Partition by userId → all events for user #456 stay in order

**3. Exactly-Once Processing**
```java
// Idempotent consumer
@KafkaListener(topics = "order-events")
public void handleOrderPlaced(OrderPlacedEvent event) {
    // Check if already processed
    if (processedEvents.contains(event.getId())) {
        return; // Skip duplicate
    }
    
    processEvent(event);
    processedEvents.add(event.getId()); // Mark as processed
}
```

**4. Dead Letter Queue (DLQ)**
```java
@KafkaListener(topics = "order-events")
public void handleOrderPlaced(OrderPlacedEvent event) {
    try {
        processEvent(event);
    } catch (Exception e) {
        // Send to DLQ for manual inspection
        deadLetterQueue.send(event, e);
        // Don't acknowledge, will retry
    }
}
```

**5. Event Sourcing (Advanced)**
- Store all events as source of truth
- Rebuild current state by replaying events
- Example: Bank account balance = sum of all Deposit/Withdraw events

```java
// Event Store
List<Event> events = eventStore.getEvents("account-123");
// [Deposit(100), Withdraw(30), Deposit(50)]
// Current balance = 100 - 30 + 50 = 120
```

### Production Tooling

**Monitoring:**
- Event lag: How far behind consumers are
- Event throughput: Events per second
- Error rates: Failed event processing
- Latency: Time from publish to consume

**Observability:**
- Distributed tracing: Follow event through services
- Correlation IDs: Link related events
- Event lineage: Which events triggered which events

---

## 6️⃣ How to implement or apply it

### Spring Boot + Kafka Implementation

**Step 1: Add Dependencies (pom.xml)**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```

**Step 2: Configure Kafka (application.yml)**
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      acks: all  # Wait for all replicas
    consumer:
      group-id: order-service-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false  # Manual commit for exactly-once
```

**Step 3: Create Event Classes**
```java
// OrderPlacedEvent.java
public class OrderPlacedEvent {
    private String orderId;
    private String userId;
    private List<OrderItem> items;
    private Instant timestamp;
    
    // Constructors, getters, setters
    public OrderPlacedEvent(String orderId, String userId, 
                           List<OrderItem> items, Instant timestamp) {
        this.orderId = orderId;
        this.userId = userId;
        this.items = items;
        this.timestamp = timestamp;
    }
}
```

**Step 4: Create Event Publisher**
```java
// EventPublisher.java
@Component
public class KafkaEventPublisher {
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    public void publish(String topic, Object event) {
        try {
            String json = objectMapper.writeValueAsString(event);
            kafkaTemplate.send(topic, json)
                .addCallback(
                    result -> log.info("Event published: {}", event),
                    failure -> log.error("Failed to publish event", failure)
                );
        } catch (Exception e) {
            log.error("Error serializing event", e);
            throw new EventPublishException("Failed to publish event", e);
        }
    }
}
```

**Step 5: Publish Events in Service**
```java
// OrderService.java
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private KafkaEventPublisher eventPublisher;
    
    public Order createOrder(OrderRequest request) {
        // 1. Create order
        Order order = new Order(
            request.getUserId(),
            request.getItems()
        );
        order = orderRepository.save(order);
        
        // 2. Publish event
        OrderPlacedEvent event = new OrderPlacedEvent(
            order.getId(),
            order.getUserId(),
            order.getItems(),
            Instant.now()
        );
        eventPublisher.publish("order-events", event);
        
        return order;
    }
}
```

**Step 6: Create Event Consumer**
```java
// OrderEventListener.java
@Component
@Slf4j
public class OrderEventListener {
    
    @Autowired
    private EmailService emailService;
    
    @KafkaListener(topics = "order-events", groupId = "email-service")
    public void handleOrderPlaced(
            @Payload String eventJson,
            Acknowledgment acknowledgment) {
        
        try {
            // 1. Deserialize
            ObjectMapper mapper = new ObjectMapper();
            OrderPlacedEvent event = mapper.readValue(
                eventJson, 
                OrderPlacedEvent.class
            );
            
            // 2. Process
            emailService.sendOrderConfirmation(
                event.getUserId(),
                event.getOrderId()
            );
            
            // 3. Acknowledge (commit offset)
            acknowledgment.acknowledge();
            
        } catch (Exception e) {
            log.error("Error processing event: {}", eventJson, e);
            // Don't acknowledge, will retry
        }
    }
}
```

**Step 7: Run Kafka Locally (docker-compose.yml)**
```yaml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      
  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

**Step 8: Create Topic**
```bash
# Start Kafka
docker-compose up -d

# Create topic
kafka-topics --create \
  --bootstrap-server localhost:9092 \
  --topic order-events \
  --partitions 3 \
  --replication-factor 1
```

**Step 9: Test**
```bash
# Start Spring Boot app
mvn spring-boot:run

# Place order (triggers event)
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": "user-1", "items": [...]}'

# Check Kafka messages
kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic order-events \
  --from-beginning
```

### Advanced: Event Sourcing

**Event Store Implementation:**
```java
// EventStore.java
@Component
public class EventStore {
    
    @Autowired
    private EventRepository eventRepository;
    
    public void save(String aggregateId, String eventType, Object payload) {
        Event event = new Event(
            aggregateId,
            eventType,
            objectMapper.writeValueAsString(payload),
            Instant.now()
        );
        eventRepository.save(event);
    }
    
    public List<Event> getEvents(String aggregateId) {
        return eventRepository.findByAggregateIdOrderByTimestamp(aggregateId);
    }
    
    // Rebuild state from events
    public Order rebuildOrder(String orderId) {
        List<Event> events = getEvents(orderId);
        Order order = new Order();
        
        for (Event event : events) {
            switch (event.getEventType()) {
                case "OrderPlaced":
                    order.applyOrderPlaced(event);
                    break;
                case "OrderCancelled":
                    order.applyOrderCancelled(event);
                    break;
                // ... more events
            }
        }
        
        return order;
    }
}
```

---

## 7️⃣ Tradeoffs, pitfalls, and common mistakes

### Tradeoffs

**Pros:**
- ✅ Loose coupling: Services don't know about each other
- ✅ Scalability: Scale consumers independently
- ✅ Resilience: One service failure doesn't cascade
- ✅ Flexibility: Add new consumers without changing producers
- ✅ Auditability: All events are logged
- ✅ Performance: Parallel processing

**Cons:**
- ❌ Complexity: More moving parts (broker, schemas, monitoring)
- ❌ Eventual consistency: Data might be temporarily inconsistent
- ❌ Debugging: Harder to trace flows across services
- ❌ Testing: Need to test event flows, not just APIs
- ❌ Duplicate events: Must handle idempotency

### Common Pitfalls

**1. Event Ordering Issues**
```java
// BAD: Events might arrive out of order
OrderCancelled arrives before OrderPlaced
→ Consumer tries to cancel non-existent order
```

**Solution:**
- Partition by aggregate ID (orderId)
- Process events in order within partition
- Use sequence numbers

**2. Lost Events**
```java
// BAD: Consumer crashes before acknowledging
Consumer processes event
→ Crashes
→ Event lost (offset not committed)
```

**Solution:**
- Idempotent processing
- Dead letter queues
- Manual offset management

**3. Event Schema Evolution**
```java
// BAD: Breaking change breaks all consumers
// Remove field "userId" → all consumers crash
```

**Solution:**
- Version events
- Backward compatible changes only
- Schema registry
- Consumer handles multiple versions

**4. Too Many Events**
```java
// BAD: Publishing events for every tiny change
UserViewedProduct → event
UserScrolledPage → event
UserMovedMouse → event
// Kafka fills up, consumers overwhelmed
```

**Solution:**
- Only publish meaningful domain events
- Batch small events
- Use different topics for different importance levels

**5. Synchronous Thinking in Async World**
```java
// BAD: Waiting for event to be processed
publishEvent(event);
waitForProcessing(); // Blocks!
return result;
```

**Solution:**
- Publish and return immediately
- Use callbacks or webhooks for results
- Accept eventual consistency

**6. Missing Idempotency**
```java
// BAD: Processing same event twice
OrderPlaced event arrives twice
→ Sends two confirmation emails
→ Charges user twice
```

**Solution:**
```java
@KafkaListener(topics = "order-events")
public void handleOrderPlaced(OrderPlacedEvent event) {
    // Check if already processed
    if (orderRepository.existsById(event.getOrderId())) {
        return; // Already processed
    }
    
    processOrder(event);
}
```

### Performance Gotchas

**1. Consumer Lag**
- Consumers fall behind producers
- Events pile up in Kafka
- Solution: Scale consumers, optimize processing

**2. Large Event Payloads**
- Sending entire objects in events
- Slow serialization/deserialization
- Solution: Send IDs, fetch data if needed (event notification)

**3. Too Many Partitions**
- Each partition needs resources
- Too many → overhead
- Too few → can't parallelize
- Solution: Start with 3-6 partitions, scale as needed

### Security Considerations

**1. Event Validation**
```java
// Validate event before processing
if (!isValidEvent(event)) {
    throw new InvalidEventException();
}
```

**2. Sensitive Data in Events**
- Events might be stored/logged
- Don't include passwords, credit cards
- Solution: Encrypt sensitive fields, or don't include them

**3. Access Control**
- Who can publish events?
- Who can consume events?
- Solution: Use Kafka ACLs, IAM policies

---

## 8️⃣ When NOT to use this

### Anti-Patterns and Misuse

**1. Simple CRUD Application**
- Single service, single database
- No need for events
- Overhead not worth it
- Use: Traditional request-response

**2. Strong Consistency Required**
- Banking: Balance must be exact immediately
- Can't accept eventual consistency
- Use: Distributed transactions (with tradeoffs)

**3. Low Latency Requirements**
- Real-time gaming: Need <10ms response
- Event processing adds latency
- Use: Direct service calls

**4. Simple Workflows**
- One service calls another
- No need for decoupling
- Events add complexity
- Use: REST/gRPC calls

**5. Small Team/System**
- 2-3 services, small team
- Event infrastructure overhead
- Use: Start with synchronous, add events later

### Signs You've Chosen Wrong

**Red Flags:**
- Events are processed synchronously (defeats purpose)
- Every API call publishes an event (too granular)
- Consumers always call back to producer (should use event-carried state)
- No event ordering strategy (chaos)
- Events used for request-response (use REST instead)

---

## 9️⃣ Comparison with Alternatives

### Event-Driven vs Request-Response

**Request-Response (REST/gRPC):**
- ✅ Simple, familiar
- ✅ Strong consistency
- ✅ Low latency (direct calls)
- ❌ Tight coupling
- ❌ Cascading failures
- ❌ Hard to scale independently

**Event-Driven:**
- ✅ Loose coupling
- ✅ Independent scaling
- ✅ Resilience
- ❌ Eventual consistency
- ❌ More complex
- ❌ Higher latency

**When to Choose:**
- Request-Response: Simple workflows, need immediate response
- Event-Driven: Complex workflows, can accept eventual consistency

### Event-Driven vs Message Queues

**Message Queues (SQS, RabbitMQ):**
- Point-to-point (one consumer per message)
- Messages deleted after consumption
- Good for: Task distribution, work queues

**Event-Driven (Kafka, EventBridge):**
- Pub/sub (multiple consumers)
- Events persist (can replay)
- Good for: Event streaming, audit logs

**When to Choose:**
- Message Queue: One consumer, fire-and-forget tasks
- Event-Driven: Multiple consumers, need replay, audit trail

### Event-Driven vs Database Replication

**Database Replication:**
- Automatic data sync
- Strong consistency
- ❌ Tight coupling (shared schema)
- ❌ Can't scale independently

**Event-Driven:**
- Services own their data
- Eventual consistency
- ✅ Loose coupling
- ✅ Independent scaling

**When to Choose:**
- Database Replication: Same data model, need strong consistency
- Event-Driven: Different data models, can accept eventual consistency

---

## 🔟 Interview follow-up questions WITH answers

### L4 (Junior) Questions

**Q1: What's the difference between event-driven and request-response?**
**A:** Request-response is synchronous: Service A calls Service B and waits for response. Event-driven is asynchronous: Service A publishes an event, and Service B (and others) react to it independently. Request-response creates tight coupling (A must know about B), while event-driven creates loose coupling (A doesn't know who listens).

**Q2: When would you use Kafka vs RabbitMQ?**
**A:** Kafka is for event streaming: multiple consumers, events persist, high throughput, replay capability. RabbitMQ is for message queues: one consumer per message, messages deleted after consumption, good for task distribution. Use Kafka when you need event history and multiple consumers. Use RabbitMQ for simple work queues.

**Q3: How do you handle duplicate events?**
**A:** Make consumers idempotent. Check if event was already processed (store event ID in database). If already processed, skip. Also use idempotency keys: same key = same operation, process only once.

### L5 (Mid-Level) Questions

**Q4: How do you ensure events are processed in order?**
**A:** Partition events by a key (like orderId). All events for the same key go to the same partition, which maintains order. Consumers process partitions sequentially. For example, partition by orderId so all OrderPlaced, OrderCancelled events for order #123 stay in order.

**Q5: What happens if a consumer crashes while processing an event?**
**A:** If consumer crashes before acknowledging, the event stays in the queue and will be redelivered. This can cause duplicate processing. Solutions: (1) Make processing idempotent, (2) Use exactly-once semantics (Kafka transactions), (3) Use dead letter queues for events that fail repeatedly.

**Q6: How do you handle schema evolution in events?**
**A:** Use a schema registry (Confluent Schema Registry). Version your events. Make changes backward compatible (add optional fields, don't remove required fields). Consumers handle multiple versions. Gradually deprecate old versions. Test compatibility before deploying.

**Q7: Event-driven architecture leads to eventual consistency. How do you handle this?**
**A:** Accept that data might be temporarily inconsistent. Use compensating actions (Saga pattern) for failures. Show users "pending" states. Use read models (CQRS) that are eventually consistent. For critical operations needing immediate consistency, use synchronous calls or distributed transactions (with tradeoffs).

### L6 (Senior) Questions

**Q8: Design an event-driven system for an e-commerce platform handling 1M orders/day.**
**A:** 
- **Event Topics:** order-events, payment-events, inventory-events, shipping-events
- **Partitioning:** Partition by orderId for ordering, by userId for user-specific events
- **Consumers:** Order service (publishes), Payment/Inventory/Shipping (consume and publish), Analytics (consumes all)
- **Scaling:** Scale consumers horizontally (add more instances)
- **Monitoring:** Track consumer lag, event throughput, error rates
- **Resilience:** Dead letter queues, retry with exponential backoff, circuit breakers
- **Consistency:** Use Saga pattern for order workflow, accept eventual consistency for analytics

**Q9: How would you implement exactly-once processing in an event-driven system?**
**A:** 
- **Idempotency:** Store processed event IDs, check before processing
- **Transactional Outbox:** Write event to database in same transaction as business logic, separate process publishes to Kafka
- **Kafka Transactions:** Use Kafka's transactional API for exactly-once semantics
- **Deduplication:** Use idempotency keys, hash event content
- **Tradeoff:** Exactly-once adds latency and complexity, often eventual consistency is acceptable

**Q10: How do you debug issues in an event-driven system?**
**A:**
- **Distributed Tracing:** Use correlation IDs to trace events across services (OpenTelemetry, Zipkin)
- **Event Logging:** Log all events with timestamps, search by correlation ID
- **Consumer Lag Monitoring:** Identify which consumers are falling behind
- **Event Replay:** Replay events to reproduce issues
- **Health Checks:** Monitor consumer health, event processing rates
- **Visualization:** Service dependency graphs, event flow diagrams

**Q11: Compare orchestration vs choreography in event-driven systems.**
**A:**
- **Orchestration:** Central coordinator (orchestrator) tells services what to do. Services don't know about each other. Example: Order orchestrator calls Payment → Inventory → Shipping sequentially.
- **Choreography:** Services react to events independently. No central coordinator. Example: OrderPlaced event triggers Payment, Inventory, Shipping services in parallel.
- **Tradeoffs:** Orchestration is easier to understand and debug, but creates a single point of failure. Choreography is more resilient but harder to understand and debug.
- **When to use:** Orchestration for complex workflows needing coordination. Choreography for simple, independent reactions.

---

## 1️⃣1️⃣ One clean mental summary

Event-Driven Architecture decouples services by having them communicate through events instead of direct calls. When something happens (like an order is placed), a service publishes an event to a message broker. Other services that care about that event subscribe to it and react independently. This allows services to work in parallel, scale independently, and remain resilient to failures. The tradeoff is eventual consistency: data might be temporarily inconsistent across services, but this is often acceptable for the benefits of loose coupling and independent scalability. Think of it like a restaurant kitchen where orders are posted on a board and different chefs work independently, rather than one chef calling another and waiting.

