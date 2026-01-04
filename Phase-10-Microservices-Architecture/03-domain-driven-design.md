# Domain-Driven Design (DDD)

## 0️⃣ Prerequisites

Before diving into this topic, you need to understand:

- **Object-Oriented Programming**: Classes, encapsulation, inheritance, polymorphism (Phase 7, Topic 1)
- **SOLID Principles**: Especially Single Responsibility and Dependency Inversion (Phase 7, Topic 2)
- **Microservices Basics**: Understanding of service boundaries and independence (Phase 10, Topic 1)
- **Database Fundamentals**: Basic understanding of data modeling and relationships
- **Event-Driven Architecture**: Domain events build on event-driven concepts (Phase 6, Topic 9)

**Quick refresher**: Microservices are independently deployable services. DDD helps you identify where to draw boundaries between services by focusing on business domains rather than technical layers.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

Imagine you're building an e-commerce system. You have these modules:

- User Management
- Product Catalog
- Shopping Cart
- Order Processing
- Payment Handling
- Inventory Management
- Shipping
- Customer Service

**The Question**: How do you split this into microservices? Where do you draw the boundaries?

**Common Anti-Pattern: Technical Layering**

Many teams split by technical layers:

```
Service 1: User Service (handles all user-related data)
Service 2: Product Service (handles all product-related data)
Service 3: Order Service (handles all order-related data)
```

This seems logical, but leads to problems:

**Problem 1: Shared Data Models**

```java
// User Service
public class User {
    private String userId;
    private String email;
    private String address;  // Needed by Shipping Service
    private String paymentMethod;  // Needed by Payment Service
    private List<Order> orders;  // Needed by Order Service
}
```

Multiple services need user data, leading to:

- Coupling (services depend on User Service)
- Inconsistency (each service has different view of "user")
- Performance issues (many calls to User Service)

**Problem 2: Business Logic Scattered**

```java
// Order Service
public void createOrder(Order order) {
    // Business rule: "Can't order if user has unpaid invoices"
    // But user invoice data is in Payment Service
    // Need to call Payment Service → coupling
}

// Payment Service
public void processPayment(Payment payment) {
    // Business rule: "Can't pay if order is cancelled"
    // But order status is in Order Service
    // Need to call Order Service → coupling
}
```

Business rules end up scattered across services, with services calling each other in circles.

**Problem 3: Poor Boundaries**

When everything is split by data entities, you get:

- Services that are too small (microservices become "nano-services")
- Services that are too large (monoliths in disguise)
- Services with unclear responsibilities

### What Systems Looked Like Before DDD

**Anemic Domain Model Anti-Pattern:**

```java
// Service Layer (all business logic here)
public class OrderService {
    public void createOrder(OrderDTO orderDto) {
        Order order = new Order();  // Just data, no behavior
        order.setUserId(orderDto.getUserId());
        order.setItems(orderDto.getItems());
        order.setStatus("PENDING");

        // Business logic in service
        if (order.getTotal() > 10000) {
            order.setRequiresApproval(true);
        }

        orderRepository.save(order);
    }
}

// Domain Model (just getters/setters)
public class Order {
    private String orderId;
    private String userId;
    private List<OrderItem> items;
    private String status;

    // Only getters and setters, no business logic
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
}
```

Problems:

- Business logic scattered in services
- Domain objects are "dumb data holders"
- Business rules duplicated across services
- Hard to understand business intent from code

### What Breaks Without DDD

**At Scale (Multiple Teams):**

```
Team A (Order Team):
  - Builds Order Service
  - Defines "order" as: orderId, userId, items, total

Team B (Shipping Team):
  - Needs order data
  - Defines "order" as: orderId, shippingAddress, weight
  - Different understanding of "order"
  - Integration becomes nightmare
```

Teams build services based on their own understanding, leading to:

- Inconsistent models
- Integration complexity
- Confusion about responsibilities

**Real Examples of the Problem:**

**Amazon (Early 2000s)**: Teams built services around technical layers (database tables). Result: services tightly coupled, changes in one service broke others, hard to scale teams.

**Netflix (Pre-DDD)**: Services split by technical concerns (User Service, Product Service). Result: business logic scattered, services calling each other in complex dependency graphs, hard to understand business flows.

**eBay (2000s)**: Monolithic codebase with anemic domain models. Business rules buried in service layers, hard to test, hard to change.

---

## 2️⃣ Intuition and Mental Model

### The City Planning Analogy

Think of **Domain-Driven Design** like city planning.

**Bad City Planning (Technical Layering):**

```
City organized by building type:
- Residential District (all houses)
- Commercial District (all shops)
- Industrial District (all factories)

Problem:
- People live in Residential, work in Commercial, commute every day
- Shops need customers from Residential
- Everything is connected, traffic is terrible
- Changing one district affects others
```

**Good City Planning (Domain-Driven):**

```
City organized by neighborhoods (bounded contexts):
- Downtown Financial District
  - Banks, offices, restaurants for workers
  - Self-contained ecosystem
- University District
  - Student housing, bookstores, cafes
  - Self-contained ecosystem
- Shopping Mall Complex
  - Stores, food court, parking
  - Self-contained ecosystem

Each neighborhood:
- Has clear boundaries
- Has its own rules and culture
- Can evolve independently
- Communicates with other neighborhoods via well-defined paths (roads)
```

### The Key Mental Model

**DDD organizes software around business domains (neighborhoods), not technical layers (building types).**

- **Bounded Context**: A neighborhood with clear boundaries
- **Domain Model**: The culture and rules of a neighborhood
- **Ubiquitous Language**: The common language used within a neighborhood
- **Context Mapping**: The roads and communication channels between neighborhoods

This analogy helps us understand:

- Why boundaries matter (neighborhoods have clear edges)
- Why models differ between contexts (downtown "customer" vs university "student")
- Why communication is explicit (well-defined roads between neighborhoods)

---

## 3️⃣ How It Works Internally

DDD consists of two main parts: **Strategic DDD** (designing boundaries) and **Tactical DDD** (implementing within boundaries).

### Strategic DDD: Designing Boundaries

**1. Domain and Subdomain**

```
E-Commerce Domain
├── Core Domain (competitive advantage)
│   └── Recommendation Engine (unique algorithm)
├── Supporting Subdomain (needed but not core)
│   ├── User Management
│   ├── Payment Processing
│   └── Shipping
└── Generic Subdomain (buy or use existing)
    ├── Email Service
    └── Analytics
```

- **Core Domain**: Where your business wins. Invest most effort here.
- **Supporting Subdomain**: Needed for business, but not differentiating.
- **Generic Subdomain**: Can use off-the-shelf solutions.

**2. Bounded Context**

A bounded context is a boundary within which a domain model applies. The same concept can have different meanings in different contexts.

```
Bounded Context: Sales
  Customer = Person who buys products
  Product = Item for sale with price

Bounded Context: Shipping
  Customer = Person to deliver to (address matters)
  Product = Package with weight and dimensions

Bounded Context: Support
  Customer = Person who needs help (support history matters)
  Product = Item they own (warranty, support ticket)
```

**3. Context Mapping**

How bounded contexts relate to each other:

```
┌──────────────────┐
│  Order Context   │───Partner──→┌──────────────────┐
│                  │             │  Payment Context │
└──────────────────┘             └──────────────────┘
        │
        │ Customer-Supplier
        ↓
┌──────────────────┐
│ Shipping Context │
│  (downstream)    │
└──────────────────┘

        │
        │ Shared Kernel (shared code)
        ↓
┌──────────────────┐
│  Common Library  │
│  (address model) │
└──────────────────┘
```

Relationship types:

- **Partnership**: Teams work together, evolve together
- **Shared Kernel**: Shared code/library between contexts
- **Customer-Supplier**: One context depends on another
- **Conformist**: Downstream context conforms to upstream model
- **Anticorruption Layer**: Translation layer between contexts
- **Separate Ways**: Independent, no integration
- **Published Language**: Well-documented integration format

### Tactical DDD: Implementing Within a Bounded Context

**1. Entity**

An object with a unique identity that persists over time.

```java
public class Order {
    private OrderId id;  // Identity
    private CustomerId customerId;
    private List<OrderItem> items;
    private OrderStatus status;

    // Identity matters, not attributes
    // Two orders with same items but different IDs are different orders
}
```

**2. Value Object**

An object defined by its attributes, not identity. Immutable.

```java
public class Money {
    private final BigDecimal amount;
    private final Currency currency;

    // No identity, defined by amount + currency
    // $100 USD is same as another $100 USD
    // Immutable: operations return new instances
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

**3. Aggregate**

A cluster of entities and value objects treated as a single unit. Has a root entity (aggregate root).

```java
// Aggregate Root
public class Order {
    private OrderId id;
    private CustomerId customerId;
    private List<OrderItem> items;  // Entities within aggregate
    private Money total;

    // Business logic in aggregate root
    public void addItem(ProductId productId, int quantity, Money price) {
        if (this.status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Cannot modify confirmed order");
        }

        OrderItem item = new OrderItem(productId, quantity, price);
        this.items.add(item);
        this.recalculateTotal();
    }

    // Only aggregate root is accessed from outside
    // OrderItem is only accessed through Order
}
```

**Aggregate Rules:**

- Only aggregate root has global identity
- External references only to aggregate root
- Consistency boundary (all or nothing)
- Small aggregates (prefer smaller)

**4. Domain Service**

Business logic that doesn't naturally fit in an entity or value object.

```java
// Domain Service
public class OrderPricingService {
    public Money calculateTotal(Order order, Customer customer) {
        Money subtotal = order.getSubtotal();

        // Business rule: VIP customers get 10% discount
        if (customer.isVip()) {
            subtotal = subtotal.multiply(new BigDecimal("0.9"));
        }

        // Business rule: Free shipping over $100
        Money shipping = Money.ZERO;
        if (subtotal.isLessThan(new Money(100, Currency.USD))) {
            shipping = new Money(10, Currency.USD);
        }

        return subtotal.add(shipping);
    }
}
```

**5. Repository**

Abstraction for accessing aggregates. Hides persistence details.

```java
public interface OrderRepository {
    Order findById(OrderId id);
    void save(Order order);
    List<Order> findByCustomerId(CustomerId customerId);
}

// Implementation uses JPA, MongoDB, etc.
// Domain doesn't know about persistence
```

**6. Domain Event**

Something that happened in the domain that other parts of the system might care about.

```java
public class OrderConfirmedEvent {
    private final OrderId orderId;
    private final CustomerId customerId;
    private final Money total;
    private final Instant occurredAt;

    // Immutable event
}

// Published when order is confirmed
public class Order {
    public void confirm() {
        // Business logic
        this.status = OrderStatus.CONFIRMED;

        // Publish domain event
        DomainEventPublisher.publish(
            new OrderConfirmedEvent(this.id, this.customerId, this.total)
        );
    }
}
```

**7. Ubiquitous Language**

A common language used by developers and domain experts.

```
Domain Expert says: "When a customer places an order, we need to reserve inventory."

Developer writes code:
- Uses term "order" not "purchase request"
- Uses term "reserve inventory" not "lock stock"
- Code reflects business language
```

---

## 4️⃣ Simulation-First Explanation

Let's build a simple e-commerce system using DDD to see how it works:

### Starting Point: Anemic Domain Model (Anti-Pattern)

```java
// OrderService.java - All logic in service
public class OrderService {
    public void createOrder(String userId, List<OrderItemDTO> items) {
        Order order = new Order();
        order.setOrderId(UUID.randomUUID().toString());
        order.setUserId(userId);
        order.setStatus("PENDING");

        // Business logic in service
        BigDecimal total = BigDecimal.ZERO;
        for (OrderItemDTO item : items) {
            total = total.add(item.getPrice().multiply(new BigDecimal(item.getQuantity())));
        }
        order.setTotal(total);

        orderRepository.save(order);
    }
}

// Order.java - Just data
public class Order {
    private String orderId;
    private String userId;
    private String status;
    private BigDecimal total;
    // Only getters/setters
}
```

**Problems:**

- Business logic in service (not in domain)
- Order is just a data container
- Hard to understand business rules from code
- Business logic duplicated if used elsewhere

### Converting to DDD: Step by Step

**Step 1: Identify Bounded Contexts**

```
Bounded Context: Order Management
  - Core concept: Order (purchase from customer)
  - Responsibilities: Create orders, manage order lifecycle
  - Language: Order, OrderItem, Customer, Product

Bounded Context: Inventory Management
  - Core concept: Stock (available items)
  - Responsibilities: Track inventory, reserve stock
  - Language: Product, Stock, Reservation

Bounded Context: Shipping
  - Core concept: Shipment (package to deliver)
  - Responsibilities: Create shipments, track delivery
  - Language: Shipment, Address, Package
```

**Step 2: Build Domain Model in Order Context**

```java
// Value Object: Money
public class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
        this.amount = amount;
        this.currency = currency;
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    // Getters, equals, hashCode...
}

// Entity: OrderItem
public class OrderItem {
    private ProductId productId;  // Reference to product in Product context
    private int quantity;
    private Money unitPrice;

    public Money getTotal() {
        return unitPrice.multiply(quantity);
    }
}

// Aggregate Root: Order
public class Order {
    private OrderId id;
    private CustomerId customerId;
    private List<OrderItem> items;
    private OrderStatus status;
    private Money total;

    // Business logic in aggregate
    public static Order create(CustomerId customerId) {
        return new Order(
            OrderId.generate(),
            customerId,
            new ArrayList<>(),
            OrderStatus.DRAFT,
            Money.ZERO
        );
    }

    public void addItem(ProductId productId, int quantity, Money unitPrice) {
        if (this.status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Cannot modify order in status: " + this.status);
        }

        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }

        OrderItem item = new OrderItem(productId, quantity, unitPrice);
        this.items.add(item);
        this.recalculateTotal();
    }

    public void confirm() {
        if (this.status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Order already confirmed");
        }

        if (this.items.isEmpty()) {
            throw new IllegalStateException("Cannot confirm empty order");
        }

        this.status = OrderStatus.CONFIRMED;

        // Publish domain event
        DomainEventPublisher.publish(
            new OrderConfirmedEvent(this.id, this.customerId, this.total)
        );
    }

    private void recalculateTotal() {
        this.total = this.items.stream()
            .map(OrderItem::getTotal)
            .reduce(Money.ZERO, Money::add);
    }

    // Getters...
}

// Domain Event
public class OrderConfirmedEvent {
    private final OrderId orderId;
    private final CustomerId customerId;
    private final Money total;
    private final Instant occurredAt;

    public OrderConfirmedEvent(OrderId orderId, CustomerId customerId, Money total) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.total = total;
        this.occurredAt = Instant.now();
    }
    // Getters...
}
```

**Step 3: Repository Interface (Domain Layer)**

```java
// Repository interface in domain
public interface OrderRepository {
    Order findById(OrderId id);
    void save(Order order);
    List<Order> findByCustomerId(CustomerId customerId);
}
```

**Step 4: Application Service (Orchestration)**

```java
// Application Service (thin orchestration layer)
@Service
public class OrderApplicationService {
    private final OrderRepository orderRepository;
    private final ProductService productService;  // From Product context

    public OrderId createOrder(CreateOrderCommand command) {
        // 1. Validate product exists (call Product context)
        for (OrderItemCommand item : command.getItems()) {
            productService.verifyProductExists(item.getProductId());
        }

        // 2. Create aggregate
        Order order = Order.create(command.getCustomerId());

        // 3. Add items
        for (OrderItemCommand item : command.getItems()) {
            Money price = productService.getPrice(item.getProductId());
            order.addItem(item.getProductId(), item.getQuantity(), price);
        }

        // 4. Save
        orderRepository.save(order);

        return order.getId();
    }

    public void confirmOrder(OrderId orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        order.confirm();  // Business logic in domain

        orderRepository.save(order);
    }
}
```

**Step 5: Handle Domain Events (Cross-Context Communication)**

```java
// In Inventory Context - Listens to OrderConfirmedEvent
@EventListener
public class InventoryEventHandler {
    public void handle(OrderConfirmedEvent event) {
        // Reserve inventory for the order
        // This is in Inventory context, uses Inventory domain model
    }
}
```

### Key Differences

**Before (Anemic):**

- Business logic in service
- Domain objects are data containers
- Hard to test business rules
- Logic scattered

**After (DDD):**

- Business logic in domain (Order aggregate)
- Domain objects have behavior
- Easy to test (domain is pure Java)
- Logic encapsulated in aggregates

---

## 5️⃣ How Engineers Actually Use This in Production

### Real-World Implementation: Spring Boot + DDD

**Project Structure:**

```
order-service/
├── src/main/java/com/company/order/
│   ├── domain/                    # Domain layer (core business logic)
│   │   ├── Order.java            # Aggregate root
│   │   ├── OrderItem.java        # Entity
│   │   ├── OrderId.java          # Value object
│   │   ├── Money.java            # Value object
│   │   ├── OrderRepository.java  # Repository interface
│   │   └── OrderConfirmedEvent.java
│   ├── application/              # Application layer (orchestration)
│   │   ├── OrderApplicationService.java
│   │   └── CreateOrderCommand.java
│   ├── infrastructure/           # Infrastructure layer (persistence, messaging)
│   │   ├── JpaOrderRepository.java
│   │   └── DomainEventPublisher.java
│   └── presentation/             # Presentation layer (REST API)
│       └── OrderController.java
```

**Domain Layer (Pure Java, No Frameworks):**

```java
// domain/Order.java
public class Order {
    // Pure Java, no Spring annotations
    // Business logic here
}

// domain/OrderRepository.java
public interface OrderRepository {
    // Interface in domain
    // Implementation in infrastructure
}
```

**Infrastructure Layer (Spring, JPA):**

```java
// infrastructure/JpaOrderRepository.java
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final SpringDataOrderRepository jpaRepo;

    @Override
    public Order findById(OrderId id) {
        return jpaRepo.findById(id)
            .map(this::toDomain)
            .orElse(null);
    }

    @Override
    public void save(Order order) {
        OrderEntity entity = toEntity(order);
        jpaRepo.save(entity);
    }

    private Order toDomain(OrderEntity entity) {
        // Map JPA entity to domain model
    }

    private OrderEntity toEntity(Order order) {
        // Map domain model to JPA entity
    }
}
```

**Application Layer (Spring Services):**

```java
// application/OrderApplicationService.java
@Service
@Transactional
public class OrderApplicationService {
    private final OrderRepository orderRepository;
    private final ProductServiceClient productService;
    private final DomainEventPublisher eventPublisher;

    public OrderId createOrder(CreateOrderCommand command) {
        // Orchestration logic
        Order order = Order.create(command.getCustomerId());
        // ...
        orderRepository.save(order);
        return order.getId();
    }
}
```

### Netflix Implementation

Netflix uses DDD principles for their microservices:

1. **Bounded Contexts**: Each microservice is a bounded context

   - Recommendation Service (core domain)
   - Playback Service
   - Billing Service
   - Content Service

2. **Domain Events**: Services communicate via events

   - User subscribed event → Billing Service
   - Content added event → Recommendation Service

3. **Ubiquitous Language**: Teams use domain language
   - "Title" not "Video"
   - "Playback" not "Streaming session"

### Amazon Implementation

Amazon's services are organized around business capabilities (bounded contexts):

1. **Product Catalog Context**: Product information, search
2. **Shopping Cart Context**: Cart management
3. **Order Management Context**: Order processing
4. **Fulfillment Context**: Warehouse, shipping
5. **Payment Context**: Payment processing

Each context has its own domain model, even though they might reference similar concepts (like "product").

### Production War Stories

**Story 1: The Shared User Model**

A company had a "User" model shared across all services. As requirements evolved:

- Order Service needed user purchase history
- Support Service needed user support tickets
- Marketing Service needed user preferences

The shared model became a "god object" with everything. Changes in one service broke others. Solution: Split into bounded contexts (Order Context has Customer, Support Context has Customer, Marketing Context has Customer) with different models, communicate via events.

**Story 2: The Anemic Order Model**

An Order service had all business logic in service classes. Rules like "orders over $1000 need approval" were scattered across multiple services. When business changed the rule to "$500", developers had to find all places and update. Solution: Moved logic into Order aggregate, single source of truth.

---

## 6️⃣ How to Implement or Apply It

### Step-by-Step: Applying DDD to a New Service

**Step 1: Event Storming (Discover Domain)**

Workshop with domain experts:

1. Write domain events on sticky notes (orange)

   - "Order Created"
   - "Payment Processed"
   - "Shipment Dispatched"

2. Write commands that trigger events (blue)

   - "Create Order" → "Order Created"
   - "Process Payment" → "Payment Processed"

3. Group related events into bounded contexts
4. Identify aggregates (clusters of events)

**Step 2: Identify Bounded Contexts**

```
From Event Storming:
- Order events cluster → Order Management Context
- Payment events cluster → Payment Context
- Shipping events cluster → Shipping Context
```

**Step 3: Define Ubiquitous Language**

Create glossary:

- Order: Customer's purchase request
- Order Item: Product and quantity in an order
- Order Status: DRAFT, CONFIRMED, SHIPPED, DELIVERED

**Step 4: Design Aggregates**

```
Order Aggregate:
- Root: Order
- Entities: OrderItem
- Value Objects: Money, OrderId, Address
- Rules: Order can only be modified in DRAFT status
```

**Step 5: Implement Domain Model**

```java
// 1. Value Objects
public class OrderId {
    private final String value;
    // Immutable, equality by value
}

public class Money {
    private final BigDecimal amount;
    private final Currency currency;
    // Immutable, business logic
}

// 2. Entities
public class OrderItem {
    private ProductId productId;
    private int quantity;
    private Money unitPrice;
    // Has identity within aggregate
}

// 3. Aggregate Root
public class Order {
    private OrderId id;
    private CustomerId customerId;
    private List<OrderItem> items;
    private OrderStatus status;

    // Business logic
    public void addItem(ProductId productId, int quantity, Money price) {
        // Business rules here
    }

    public void confirm() {
        // Business logic + domain event
    }
}
```

**Step 6: Repository Interface**

```java
public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    void save(Order order);
    List<Order> findByCustomerId(CustomerId customerId);
}
```

**Step 7: Application Service**

```java
@Service
@Transactional
public class OrderApplicationService {
    private final OrderRepository orderRepository;
    private final ProductServiceClient productService;

    public OrderId createOrder(CreateOrderCommand command) {
        // Orchestration
        Order order = Order.create(command.getCustomerId());

        for (OrderItemCommand item : command.getItems()) {
            Money price = productService.getPrice(item.getProductId());
            order.addItem(item.getProductId(), item.getQuantity(), price);
        }

        orderRepository.save(order);
        return order.getId();
    }
}
```

**Step 8: Infrastructure Implementation**

```java
// JPA Entity (separate from domain model)
@Entity
@Table(name = "orders")
public class OrderEntity {
    @Id
    private String id;
    private String customerId;
    private String status;
    // JPA annotations, no business logic
}

// Repository Implementation
@Repository
public class JpaOrderRepository implements OrderRepository {
    @Autowired
    private SpringDataOrderRepository jpaRepo;

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepo.findById(id.getValue())
            .map(this::toDomain);
    }

    private Order toDomain(OrderEntity entity) {
        // Map JPA → Domain
        return Order.reconstitute(
            OrderId.of(entity.getId()),
            CustomerId.of(entity.getCustomerId()),
            // ...
        );
    }
}
```

**Step 9: REST Controller**

```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    private final OrderApplicationService orderService;

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request) {
        CreateOrderCommand command = toCommand(request);
        OrderId orderId = orderService.createOrder(command);
        return ResponseEntity.ok(new OrderResponse(orderId.getValue()));
    }
}
```

### Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <!-- Domain layer has no dependencies -->
</dependencies>
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Tradeoffs

**Advantages:**

- **Clear Boundaries**: Bounded contexts create clear service boundaries
- **Business Alignment**: Code reflects business language and rules
- **Testability**: Domain logic is pure Java, easy to test
- **Maintainability**: Business logic centralized in aggregates
- **Team Scalability**: Teams can work independently in their contexts

**Disadvantages:**

- **Complexity**: More layers, more abstractions
- **Learning Curve**: Team needs to understand DDD concepts
- **Over-Engineering Risk**: Can be overkill for simple CRUD apps
- **Performance**: Domain model → Persistence mapping adds overhead
- **Time Investment**: Event storming and modeling take time

### Common Pitfalls

**Pitfall 1: Anemic Domain Model**

```java
// BAD: Domain objects are data containers
public class Order {
    private String status;
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    // No business logic
}
```

**Solution**: Move business logic into domain objects.

**Pitfall 2: God Aggregates**

```java
// BAD: Aggregate that's too large
public class Order {
    private List<OrderItem> items;
    private Payment payment;
    private Shipment shipment;
    private Customer customer;  // Should be reference, not part of aggregate
    // Too many responsibilities
}
```

**Solution**: Keep aggregates small. Use references (IDs) to other aggregates.

**Pitfall 3: Leaky Abstractions**

```java
// BAD: Infrastructure concerns in domain
public class Order {
    @Entity  // JPA annotation in domain
    @Id
    @GeneratedValue
    private Long id;  // Database ID, not domain ID
}
```

**Solution**: Separate domain model from persistence model. Domain has no framework annotations.

**Pitfall 4: Shared Database**

```
Order Service ──→ Database ←── Payment Service
                    │
                    └── Shipping Service
```

Multiple services sharing same database breaks bounded contexts.

**Solution**: Each bounded context has its own database.

**Pitfall 5: Ignoring Ubiquitous Language**

```java
// BAD: Technical terms
public class OrderDTO {
    private String purchaseRequestId;  // Business says "order", not "purchase request"
}
```

**Solution**: Use business language in code.

### Performance Considerations

**Aggregate Size:**

Large aggregates load all data even if you only need part:

```java
// BAD: Large aggregate
public class Order {
    private List<OrderItem> items;  // 1000 items
    private List<Payment> payments;  // Loaded even if not needed
}

// GOOD: Keep aggregates small, lazy load if needed
```

**N+1 Queries:**

```java
// BAD: Loading related aggregates one by one
for (Order order : orders) {
    Customer customer = customerRepo.findById(order.getCustomerId());  // N queries
}
```

**Solution**: Use application service to batch load, or use read models (CQRS).

---

## 8️⃣ When NOT to Use This

### Anti-Patterns and Misuse Cases

**Don't use DDD for:**

**1. Simple CRUD Applications**

If your application is just Create, Read, Update, Delete with no complex business rules, DDD is overkill. Use simple data models.

**2. Data-Intensive Applications**

Applications focused on data processing, analytics, reporting might not benefit from DDD. Focus on data modeling instead.

**3. Proof of Concepts**

For rapid prototyping, DDD adds unnecessary ceremony. Start simple, introduce DDD if it becomes production.

### Situations Where This is Overkill

**Small Team, Simple Domain:**

If you have 2-3 developers working on a simple application, the overhead of DDD (event storming, bounded contexts, aggregates) might not be worth it.

**Legacy System Integration:**

When integrating with legacy systems, sometimes it's better to create an anticorruption layer rather than full DDD.

### Better Alternatives for Specific Scenarios

**Transaction Script Pattern:**

For simple workflows:

```java
// Simple transaction script
public void processOrder(OrderData data) {
    // Step 1: Validate
    // Step 2: Save to database
    // Step 3: Send email
    // All in one method
}
```

**Active Record Pattern:**

For simple domain models:

```java
// Active Record (Rails style)
public class Order extends ActiveRecord<Order> {
    public void confirm() {
        this.status = "CONFIRMED";
        this.save();
    }
}
```

### Signs You've Chosen Wrong Approach

- Spending more time on modeling than building features
- Business logic still scattered despite DDD
- Team confused about boundaries
- Simple changes require changing multiple layers
- Performance issues from over-abstraction

---

## 9️⃣ Comparison with Alternatives

### DDD vs Transaction Script

**Transaction Script:**

```
Service Layer:
  - All logic in service methods
  - Database-centric
  - Good for simple workflows
```

**DDD:**

```
Domain Layer:
  - Logic in domain objects
  - Domain-centric
  - Good for complex business rules
```

**When to choose DDD**: Complex business logic, rich domain models
**When to choose Transaction Script**: Simple CRUD, straightforward workflows

### DDD vs Anemic Domain Model

**Anemic Domain Model:**

```java
// Data + Logic separated
public class Order {
    // Only data
}
public class OrderService {
    // All logic
}
```

**DDD:**

```java
// Data + Logic together
public class Order {
    // Data + behavior
    public void confirm() { /* logic */ }
}
```

**DDD is preferred** for complex domains because logic is co-located with data, easier to understand and test.

### Strategic DDD vs Tactical DDD

**Strategic DDD** (boundary design):

- Bounded contexts
- Context mapping
- Core/supporting/generic subdomains
- Use when designing microservices

**Tactical DDD** (implementation):

- Aggregates, entities, value objects
- Repositories, domain services
- Use when implementing within a service

**You can use one without the other**, but they work best together.

---

## 🔟 Interview Follow-up Questions WITH Answers

### L4 Level Questions

**Q1: What is Domain-Driven Design?**

**Answer**: Domain-Driven Design (DDD) is an approach to software development that centers the development on programming a domain model that has a rich understanding of the processes and rules of a domain. It focuses on:

- Organizing code around business domains (bounded contexts) rather than technical layers
- Using ubiquitous language (business terms in code)
- Implementing rich domain models with business logic in domain objects
- Strategic design (boundaries) and tactical design (implementation patterns)

**Q2: What is a bounded context?**

**Answer**: A bounded context is a boundary within which a particular domain model is valid. The same concept (like "customer" or "product") can have different meanings and models in different bounded contexts. For example, in an Order context, "customer" might mean someone who places orders, while in a Support context, "customer" might mean someone who opens support tickets. Each context has its own model and language.

**Q3: What is the difference between an entity and a value object?**

**Answer**:

- **Entity**: Has a unique identity that persists over time. Two entities with the same attributes but different IDs are different objects. Example: Order (Order #123 is different from Order #124 even if they have same items).

- **Value Object**: Defined entirely by its attributes. No identity. Two value objects with the same attributes are considered equal. Usually immutable. Example: Money ($100 USD is the same as another $100 USD).

### L5 Level Questions

**Q4: What is an aggregate in DDD?**

**Answer**: An aggregate is a cluster of entities and value objects treated as a single unit. It has:

- **Aggregate Root**: The single entity that external code can hold references to. Only the root has global identity.
- **Consistency Boundary**: All changes within an aggregate are consistent (all or nothing).
- **Invariants**: Business rules that must always be true within the aggregate.

Example: Order aggregate with Order (root) and OrderItems (entities within aggregate). External code only references Order, not OrderItem directly.

**Q5: How do bounded contexts communicate in DDD?**

**Answer**: Bounded contexts communicate through:

1. **Domain Events**: When something important happens in one context, it publishes an event. Other contexts listen and react.
2. **Shared Kernel**: Shared code/library (use sparingly, creates coupling).
3. **Published Language**: Well-documented integration format (APIs, message schemas).
4. **Anticorruption Layer**: Translation layer that converts one context's model to another's.

Best practice: Prefer domain events for asynchronous communication. Avoid shared databases (breaks bounded context boundaries).

**Q6: Explain the difference between strategic and tactical DDD.**

**Answer**:

- **Strategic DDD**: Focuses on boundaries and relationships between bounded contexts. Includes: bounded contexts, context mapping (how contexts relate), identifying core/supporting/generic subdomains. Used when designing system architecture and microservice boundaries.

- **Tactical DDD**: Focuses on implementation within a bounded context. Includes: aggregates, entities, value objects, repositories, domain services, domain events. Used when implementing the domain model.

Strategic DDD answers "where are the boundaries?" Tactical DDD answers "how do I implement within a boundary?"

### L6 Level Questions

**Q7: You're designing a microservices system for an e-commerce platform. How do you use DDD to identify service boundaries?**

**Answer**:

1. **Event Storming**: Workshop with domain experts to identify domain events (Order Created, Payment Processed, Shipment Dispatched).

2. **Identify Bounded Contexts**: Group related events. Example:

   - Order events → Order Management Context
   - Payment events → Payment Context
   - Shipping events → Fulfillment Context
   - Product events → Catalog Context

3. **Context Mapping**: Understand relationships:

   - Order → Payment: Customer-Supplier (Order needs payment)
   - Order → Fulfillment: Customer-Supplier (Order triggers fulfillment)
   - Catalog → Order: Published Language (Order references product IDs)

4. **Identify Core Domain**: What's your competitive advantage? (e.g., Recommendation Engine) → Invest most here.

5. **Define Service Boundaries**: Each bounded context becomes a microservice (or a group of related microservices).

6. **Design Integration**: Use domain events for cross-context communication. Order Service publishes OrderConfirmedEvent, Payment and Fulfillment services listen.

**Q8: How do you handle transactions that span multiple aggregates in DDD?**

**Answer**: In DDD, you typically **don't** have transactions that span aggregates. Each aggregate is a consistency boundary. However, you need eventual consistency across aggregates:

1. **Domain Events**: When Aggregate A completes its operation, it publishes a domain event. Aggregate B listens and processes asynchronously.

Example:

- Order aggregate: `order.confirm()` → publishes OrderConfirmedEvent
- Inventory aggregate: Listens to event → reserves inventory (separate transaction)
- If inventory reservation fails, use compensating actions (Saga pattern)

2. **Saga Pattern**: For workflows spanning multiple aggregates:

   - Orchestration: Central coordinator (orchestrator) coordinates steps
   - Choreography: Each step publishes events, next step listens

3. **Two-Phase Commit Avoided**: DDD avoids distributed transactions (2PC) because they create coupling and performance issues. Prefer eventual consistency with domain events.

**Q9: You have an existing monolith with an anemic domain model. How do you refactor it to use DDD?**

**Answer**:

1. **Identify Bounded Contexts**: Analyze existing modules, group by business capability (not technical layers).

2. **Start with One Context**: Don't refactor everything at once. Start with one bounded context (usually core domain or most problematic area).

3. **Extract Domain Model**:

   - Identify entities and value objects
   - Move business logic from service classes to domain objects
   - Create aggregates (identify aggregate roots)

4. **Introduce Repository Pattern**:

   - Create repository interfaces in domain
   - Implement with existing persistence (JPA, etc.)
   - Gradually separate domain model from persistence model

5. **Introduce Domain Events**:

   - Identify important domain events
   - Publish events from aggregates
   - Update other parts to listen to events (gradually decouple)

6. **Strangler Pattern**: Gradually replace monolith parts with DDD-based services:

   - Create new service with DDD
   - Route new features to new service
   - Migrate existing features gradually
   - Eventually retire monolith

7. **Team Education**: Train team on DDD concepts, ubiquitous language, aggregate design.

Key: Incremental migration, not big-bang rewrite.

---

## 1️⃣1️⃣ One Clean Mental Summary

Domain-Driven Design organizes software around business domains (bounded contexts) rather than technical layers. Think of it like city planning: organize by neighborhoods (business contexts) with clear boundaries, not by building types (technical layers). Within each neighborhood (bounded context), you build rich domain models where business logic lives in domain objects (aggregates, entities, value objects), not scattered in service layers. The same concept can have different meanings in different contexts (a "customer" in Order context is different from "customer" in Support context). Contexts communicate via domain events and well-defined interfaces. DDD helps identify microservice boundaries, makes code reflect business language, and centralizes business logic in testable domain objects. Use strategic DDD to design boundaries, tactical DDD to implement within them.
