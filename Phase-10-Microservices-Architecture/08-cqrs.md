# CQRS (Command Query Responsibility Segregation)

## 0️⃣ Prerequisites

Before diving into this topic, you need to understand:

- **Database per Service**: Why services have separate databases (Phase 10, Topic 7)
- **Event-Driven Architecture**: Events, event sourcing basics (Phase 6, Topic 14)
- **CAP Theorem**: Consistency vs Availability trade-offs (Phase 1)
- **Database Fundamentals**: Read vs Write operations, indexes (Phase 1)
- **Microservices Communication**: How services communicate (Phase 10, Topic 6)

**Quick refresher**: In traditional systems, the same database model handles both reads and writes. CQRS separates these: one model for writes (commands) and a different model for reads (queries). This allows optimizing each for its specific purpose.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

In traditional systems, you use the same database model for both reading and writing:

```
Traditional System
├── User creates order (WRITE)
│   └── INSERT INTO orders (id, user_id, total, status, created_at, ...)
│
└── User views order history (READ)
    └── SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC
```

**The Problem: Read and Write Have Different Requirements**

**Write Requirements:**
- Normalized data (avoid duplication)
- Enforce business rules
- Maintain consistency
- Optimize for writes (fast inserts/updates)

**Read Requirements:**
- Denormalized data (join everything for display)
- Fast queries (indexes, materialized views)
- Optimize for reads (fast selects)
- Different shapes for different views (list view vs detail view)

**The Conflict:**
- Optimizing for writes hurts reads (normalized = many joins)
- Optimizing for reads hurts writes (denormalized = slow updates)
- One model can't be optimal for both

### Real-World Example: E-commerce Order System

**Traditional Approach (Single Model):**

```sql
-- Write: Create order
INSERT INTO orders (id, user_id, total, status)
VALUES (1, 123, 100.00, 'PENDING');

-- Read: Show order history page
SELECT 
    o.id,
    o.total,
    o.status,
    u.name as user_name,           -- Join users table
    u.email as user_email,          -- Join users table
    p.name as product_name,         -- Join order_items, then products
    p.price as product_price        -- Join order_items, then products
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.user_id = 123
ORDER BY o.created_at DESC;
```

**Problems:**
- Slow query (multiple joins)
- Complex query (hard to optimize)
- Can't cache easily (data changes frequently)
- Hard to scale reads independently

**CQRS Approach:**

```sql
-- Write Model: Normalized, optimized for writes
INSERT INTO orders (id, user_id, total, status)
VALUES (1, 123, 100.00, 'PENDING');

-- Read Model: Denormalized, optimized for reads
INSERT INTO order_read_model (
    order_id,
    user_name,      -- Denormalized
    user_email,     -- Denormalized
    product_names,  -- Denormalized (JSON array)
    total,
    status
) VALUES (
    1,
    'John Doe',     -- Copied from users table
    'john@example.com',
    '["Product A", "Product B"]',
    100.00,
    'PENDING'
);

-- Read: Fast, simple query
SELECT * FROM order_read_model
WHERE user_id = 123
ORDER BY created_at DESC;
```

**Benefits:**
- Fast reads (no joins, pre-aggregated)
- Can scale reads independently (separate database)
- Can cache easily (read model changes less frequently)
- Different read models for different views

### What Breaks Without CQRS

**Scenario 1: High Read Load**
- Millions of users viewing order history
- Single database struggles with read queries
- Can't scale reads without scaling writes (they share database)
- Solution: Separate read database, scale it independently

**Scenario 2: Complex Reporting Queries**
- Business needs: "Revenue by product, by month, by region"
- Requires joins across orders, products, users, regions
- Slow queries block write operations
- Solution: Pre-compute reports in read model

**Scenario 3: Different Views Need Different Data Shapes**
- Mobile app needs: order_id, status, total (simple)
- Web app needs: full order details with all relationships
- Admin dashboard needs: aggregated statistics
- Single model can't optimize for all
- Solution: Different read models for each view

### Real Examples

**Netflix**: 
- Write: User watches video (simple write to event log)
- Read: Recommendation engine needs complex aggregated data (separate read model)

**Uber**:
- Write: Driver updates location (simple write)
- Read: Map view needs aggregated location data (separate read model)

**Amazon**:
- Write: Order created (normalized, ACID)
- Read: Order history page (denormalized, fast queries)

---

## 2️⃣ Intuition and Mental Model

### The Library Analogy

**Traditional System (Single Model)** is like a library with one catalog:
- Librarian writes new book entry (WRITE)
- Patron searches catalog (READ)
- Same catalog used for both
- Problem: Optimizing catalog for writing (fast to add books) makes searching slow
- Problem: Optimizing catalog for reading (detailed indexes) makes writing slow

**CQRS** is like a library with two systems:
- **Write System**: Librarian's catalog (normalized, structured, for adding books)
  - Fast to add new books
  - Enforces rules (ISBN must be unique)
  - Maintains consistency
  
- **Read System**: Public search system (denormalized, indexed, for finding books)
  - Fast to search
  - Pre-computed indexes
  - Different views (by author, by genre, by popularity)
  - Updated from write system (eventually)

**The Flow:**
1. Librarian adds book to write catalog
2. System automatically updates read catalog (search system)
3. Patrons search read catalog (fast, optimized)
4. If read catalog is slightly behind, that's OK (eventual consistency)

### The Restaurant Menu Analogy

**Traditional**: Chef writes menu on whiteboard, customers read same whiteboard
- Chef needs to write fast (add/remove items)
- Customers need to read easily (organized by category, with prices)
- Conflict: Organized menu is slow to update

**CQRS**: 
- **Write Model**: Chef's notebook (simple list, easy to update)
- **Read Model**: Printed menu (beautiful, organized, optimized for reading)
- When chef updates notebook, menu gets reprinted (eventually)

---

## 3️⃣ How It Works Internally

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Command Side (Write)                  │
│                                                          │
│  Client → Command → Command Handler → Write Database    │
│              │              │                │           │
│              │              │                │           │
│              └──────────────┴────────────────┘           │
│                           │                              │
│                    Domain Events                         │
│                           │                              │
└───────────────────────────┼──────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                     Query Side (Read)                    │
│                                                          │
│  Event Handler → Read Model Updater → Read Database     │
│                                                          │
│  Client → Query → Query Handler → Read Database         │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Key Components

**1. Command Side (Write Model)**
- **Commands**: "CreateOrder", "CancelOrder", "UpdateOrderStatus"
- **Command Handler**: Validates, applies business rules, updates write database
- **Write Database**: Normalized, optimized for writes, source of truth
- **Domain Events**: Published when commands succeed

**2. Query Side (Read Model)**
- **Queries**: "GetOrderHistory", "GetOrderDetails", "GetOrderStatistics"
- **Query Handler**: Reads from read database (no business logic)
- **Read Database**: Denormalized, optimized for reads, can have multiple models
- **Event Handler**: Listens to domain events, updates read model

### Data Flow: Creating an Order

**Step 1: Command Received**
```
Client sends: CreateOrderCommand {
    userId: 123,
    items: [{productId: "P1", quantity: 2}],
    total: 100.00
}
```

**Step 2: Command Handler Processes**
```java
// Write side
CommandHandler.handle(CreateOrderCommand cmd) {
    // Validate business rules
    validateUserExists(cmd.userId);
    validateStockAvailable(cmd.items);
    
    // Create order in write database
    Order order = new Order(cmd.userId, cmd.items, cmd.total);
    writeRepository.save(order);
    
    // Publish event
    eventPublisher.publish(new OrderCreatedEvent(order));
}
```

**Step 3: Write Database Updated**
```sql
-- Write database (normalized)
INSERT INTO orders (id, user_id, total, status, created_at)
VALUES (1, 123, 100.00, 'PENDING', NOW());

INSERT INTO order_items (order_id, product_id, quantity, price)
VALUES (1, 'P1', 2, 50.00);
```

**Step 4: Event Published**
```
OrderCreatedEvent {
    orderId: 1,
    userId: 123,
    items: [...],
    total: 100.00,
    timestamp: 2024-01-15T10:00:00Z
}
```

**Step 5: Read Model Updated (Asynchronously)**
```java
// Read side
EventHandler.handle(OrderCreatedEvent event) {
    // Get additional data needed for read model
    User user = userService.getUser(event.userId);
    List<Product> products = productService.getProducts(event.items);
    
    // Create denormalized read model
    OrderReadModel readModel = new OrderReadModel(
        event.orderId,
        user.getName(),           // Denormalized
        user.getEmail(),          // Denormalized
        products.stream().map(Product::getName).collect(...), // Denormalized
        event.total,
        event.status
    );
    
    readRepository.save(readModel);
}
```

**Step 6: Read Database Updated**
```sql
-- Read database (denormalized)
INSERT INTO order_read_model (
    order_id,
    user_id,
    user_name,        -- Denormalized from users table
    user_email,       -- Denormalized from users table
    product_names,    -- Denormalized JSON: ["Product A", "Product B"]
    total,
    status,
    created_at
) VALUES (
    1,
    123,
    'John Doe',
    'john@example.com',
    '["Product A", "Product B"]',
    100.00,
    'PENDING',
    '2024-01-15 10:00:00'
);
```

**Step 7: Query Served from Read Model**
```java
// Query side
QueryHandler.handle(GetOrderHistoryQuery query) {
    // Simple, fast query (no joins)
    return readRepository.findByUserIdOrderByCreatedAtDesc(query.userId);
}
```

```sql
-- Fast read query (no joins needed)
SELECT * FROM order_read_model
WHERE user_id = 123
ORDER BY created_at DESC;
```

### Key Insight: Eventual Consistency

**Important**: Read model is updated asynchronously. There's a delay between:
- Write completing (order created in write database)
- Read model updating (order appears in read database)

**This is OK because:**
- Reads don't need to be immediately consistent
- User can refresh page if data seems stale
- Write is the source of truth

**If you need immediate consistency:**
- Read from write model for that specific query
- Or use synchronous read model update (slower, but consistent)

---

## 4️⃣ Simulation-First Explanation

### Simple Example: Order System

**Setup:**
- **Write Model**: PostgreSQL (normalized, ACID)
- **Read Model**: PostgreSQL (denormalized, optimized for queries)
- **Event Bus**: Kafka (or in-memory for simple case)

### Scenario: User Creates Order

**Initial State:**
```
Write Database:
  orders: empty
  order_items: empty

Read Database:
  order_read_model: empty
```

**Step 1: User Submits Order**
```
POST /api/orders
{
  "userId": 123,
  "items": [{"productId": "P1", "quantity": 2}],
  "total": 100.00
}
```

**Step 2: Command Handler**
```java
@CommandHandler
public void handle(CreateOrderCommand cmd) {
    // Business logic
    if (cmd.total < 0) {
        throw new InvalidOrderException();
    }
    
    // Write to write database
    Order order = new Order();
    order.setUserId(cmd.userId);
    order.setTotal(cmd.total);
    order.setStatus("PENDING");
    writeOrderRepository.save(order);
    
    // Save order items
    for (OrderItem item : cmd.items) {
        OrderItemEntity itemEntity = new OrderItemEntity();
        itemEntity.setOrderId(order.getId());
        itemEntity.setProductId(item.getProductId());
        itemEntity.setQuantity(item.getQuantity());
        writeOrderItemRepository.save(itemEntity);
    }
    
    // Publish event
    OrderCreatedEvent event = new OrderCreatedEvent(
        order.getId(),
        cmd.userId,
        cmd.items,
        cmd.total
    );
    eventPublisher.publish(event);
    
    // Return immediately (don't wait for read model)
    return order.getId();
}
```

**Step 3: Write Database State**
```sql
-- Write database
orders table:
  id: 1
  user_id: 123
  total: 100.00
  status: 'PENDING'
  created_at: 2024-01-15 10:00:00

order_items table:
  order_id: 1
  product_id: 'P1'
  quantity: 2
  price: 50.00
```

**Step 4: Event Published to Kafka**
```
Topic: order-events
Message: {
  "type": "OrderCreated",
  "orderId": 1,
  "userId": 123,
  "items": [{"productId": "P1", "quantity": 2}],
  "total": 100.00,
  "timestamp": "2024-01-15T10:00:00Z"
}
```

**Step 5: Event Handler (Read Side)**
```java
@EventListener
public void handle(OrderCreatedEvent event) {
    // Fetch additional data needed for read model
    User user = userServiceClient.getUser(event.getUserId());
    
    List<String> productNames = new ArrayList<>();
    for (OrderItem item : event.getItems()) {
        Product product = productServiceClient.getProduct(item.getProductId());
        productNames.add(product.getName());
    }
    
    // Create read model
    OrderReadModel readModel = new OrderReadModel();
    readModel.setOrderId(event.getOrderId());
    readModel.setUserId(event.getUserId());
    readModel.setUserName(user.getName());           // Denormalized
    readModel.setUserEmail(user.getEmail());         // Denormalized
    readModel.setProductNames(productNames);         // Denormalized
    readModel.setTotal(event.getTotal());
    readModel.setStatus(event.getStatus());
    readModel.setCreatedAt(event.getTimestamp());
    
    // Save to read database
    readOrderRepository.save(readModel);
}
```

**Step 6: Read Database State**
```sql
-- Read database
order_read_model table:
  order_id: 1
  user_id: 123
  user_name: 'John Doe'              -- Denormalized
  user_email: 'john@example.com'      -- Denormalized
  product_names: '["Product A"]'      -- Denormalized JSON
  total: 100.00
  status: 'PENDING'
  created_at: 2024-01-15 10:00:00
```

**Step 7: User Queries Order History**
```
GET /api/orders/history?userId=123
```

**Step 8: Query Handler (Read Side)**
```java
@QueryHandler
public List<OrderReadModel> handle(GetOrderHistoryQuery query) {
    // Simple, fast query (no joins, no business logic)
    return readOrderRepository.findByUserIdOrderByCreatedAtDesc(
        query.getUserId()
    );
}
```

```sql
-- Fast read query
SELECT * FROM order_read_model
WHERE user_id = 123
ORDER BY created_at DESC;
-- No joins needed! All data is already denormalized.
```

### Timeline

```
T=0ms:   User submits order
T=10ms:  Order saved to write database
T=11ms:  Event published
T=12ms:  Response returned to user (order ID)
T=50ms:  Event handler processes event
T=100ms: Read model updated
T=200ms: User queries order history → reads from read model (fast!)
```

**Notice:**
- User gets response immediately (T=12ms)
- Read model updates later (T=100ms)
- If user queries immediately, might not see order (eventual consistency)
- Usually fine: user refreshes page, sees order

---

## 5️⃣ How Engineers Actually Use This in Production

### Netflix: Recommendation System

**Write Side:**
- User watches video → Event: "VideoWatched"
- Simple write to event log
- Fast, no complex processing

**Read Side:**
- Recommendation engine needs: "Users who watched X also watched Y"
- Complex aggregation across millions of events
- Pre-computed in read model (updated asynchronously)
- Serves recommendations instantly

**Why CQRS:**
- Write: Simple event (fast)
- Read: Complex aggregation (pre-computed, fast to serve)

### Uber: Trip History

**Write Side:**
- Driver updates location → Event: "LocationUpdated"
- Simple write (high frequency, millions per second)
- Optimized for writes

**Read Side:**
- User views trip history
- Needs: trip details, driver info, route, price breakdown
- Denormalized read model (all data in one place)
- Fast queries, no joins

**Why CQRS:**
- Write: Simple location update (optimize for throughput)
- Read: Complex trip view (optimize for query speed)

### Amazon: Order System

**Write Side:**
- Order created → Normalized database
- Enforces business rules
- ACID transactions

**Read Side:**
- Order history page
- Needs: order + user info + product info + shipping info
- Denormalized read model
- Fast page loads

**Why CQRS:**
- Write: Need consistency (can't lose orders)
- Read: Need speed (users expect fast page loads)

### Twitter/X: Timeline

**Write Side:**
- User posts tweet → Simple write
- Fast, high throughput

**Read Side:**
- User views timeline
- Needs: tweets from followed users, with user info, engagement metrics
- Complex aggregation (millions of tweets)
- Pre-computed timeline in read model

**Why CQRS:**
- Write: Simple tweet creation (optimize for writes)
- Read: Complex timeline generation (pre-compute, serve fast)

---

## 6️⃣ How to Implement This

### Example: Order System with CQRS

### Step 1: Project Structure

```
order-service/
├── src/main/java/
│   ├── command/              # Write side
│   │   ├── CreateOrderCommand.java
│   │   ├── CancelOrderCommand.java
│   │   └── CommandHandler.java
│   ├── query/                # Read side
│   │   ├── GetOrderQuery.java
│   │   ├── GetOrderHistoryQuery.java
│   │   └── QueryHandler.java
│   ├── event/                # Events
│   │   ├── OrderCreatedEvent.java
│   │   └── EventHandler.java
│   ├── write/                # Write model
│   │   ├── Order.java (entity)
│   │   └── OrderRepository.java
│   └── read/                 # Read model
│       ├── OrderReadModel.java (entity)
│       └── OrderReadModelRepository.java
```

### Step 2: Dependencies (pom.xml)

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
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
</dependencies>
```

### Step 3: Write Model (Command Side)

**Order Entity (Write Model):**
```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private Long userId;
    private BigDecimal total;
    private String status;
    private LocalDateTime createdAt;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items;
    
    // Normalized: references to other entities
    // No denormalized data here
    
    // Getters and setters
}
```

**OrderItem Entity:**
```java
@Entity
@Table(name = "order_items")
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;
    
    private String productId;
    private Integer quantity;
    private BigDecimal price;
    
    // Normalized: references product by ID
    // Getters and setters
}
```

**CreateOrderCommand:**
```java
public class CreateOrderCommand {
    private Long userId;
    private List<OrderItemRequest> items;
    private BigDecimal total;
    
    // Getters and setters
}

public class OrderItemRequest {
    private String productId;
    private Integer quantity;
    private BigDecimal price;
}
```

**Command Handler:**
```java
@Service
public class OrderCommandHandler {
    
    @Autowired
    private OrderWriteRepository orderRepository;
    
    @Autowired
    private EventPublisher eventPublisher;
    
    @Transactional
    public Long handle(CreateOrderCommand command) {
        // Business logic validation
        validateCommand(command);
        
        // Create order in write database
        Order order = new Order();
        order.setUserId(command.getUserId());
        order.setTotal(command.getTotal());
        order.setStatus("PENDING");
        order.setCreatedAt(LocalDateTime.now());
        
        List<OrderItem> items = command.getItems().stream()
            .map(item -> {
                OrderItem orderItem = new OrderItem();
                orderItem.setProductId(item.getProductId());
                orderItem.setQuantity(item.getQuantity());
                orderItem.setPrice(item.getPrice());
                orderItem.setOrder(order);
                return orderItem;
            })
            .collect(Collectors.toList());
        
        order.setItems(items);
        order = orderRepository.save(order);
        
        // Publish event (asynchronously updates read model)
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getUserId(),
            command.getItems(),
            order.getTotal(),
            order.getStatus(),
            order.getCreatedAt()
        );
        eventPublisher.publish(event);
        
        return order.getId();
    }
    
    private void validateCommand(CreateOrderCommand command) {
        if (command.getTotal().compareTo(BigDecimal.ZERO) < 0) {
            throw new InvalidOrderException("Total cannot be negative");
        }
        // More validation...
    }
}
```

**Write Repository:**
```java
@Repository
public interface OrderWriteRepository extends JpaRepository<Order, Long> {
    // Write-side queries (if needed)
    // Usually minimal, just CRUD
}
```

### Step 4: Event

**OrderCreatedEvent:**
```java
public class OrderCreatedEvent {
    private Long orderId;
    private Long userId;
    private List<OrderItemEvent> items;
    private BigDecimal total;
    private String status;
    private LocalDateTime createdAt;
    
    // Getters and setters
    // Serializable for Kafka
}
```

**Event Publisher:**
```java
@Service
public class EventPublisher {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void publish(OrderCreatedEvent event) {
        kafkaTemplate.send("order-events", event);
    }
}
```

### Step 5: Read Model (Query Side)

**OrderReadModel Entity:**
```java
@Entity
@Table(name = "order_read_model")
public class OrderReadModel {
    @Id
    private Long orderId;  // Same ID as write model
    
    private Long userId;
    
    // Denormalized data (copied from other services/tables)
    private String userName;        // From User Service
    private String userEmail;       // From User Service
    private String productNames;    // JSON array: ["Product A", "Product B"]
    private String productIds;      // JSON array: ["P1", "P2"]
    
    private BigDecimal total;
    private String status;
    private LocalDateTime createdAt;
    
    // All data needed for queries in one place
    // No joins needed!
    
    // Getters and setters
}
```

**Event Handler (Updates Read Model):**
```java
@Service
public class OrderEventHandler {
    
    @Autowired
    private OrderReadModelRepository readRepository;
    
    @Autowired
    private UserServiceClient userServiceClient;
    
    @Autowired
    private ProductServiceClient productServiceClient;
    
    @KafkaListener(topics = "order-events")
    public void handle(OrderCreatedEvent event) {
        // Fetch additional data needed for read model
        UserDTO user = userServiceClient.getUser(event.getUserId());
        
        List<String> productNames = new ArrayList<>();
        List<String> productIds = new ArrayList<>();
        
        for (OrderItemEvent item : event.getItems()) {
            ProductDTO product = productServiceClient.getProduct(item.getProductId());
            productNames.add(product.getName());
            productIds.add(product.getId());
        }
        
        // Create/update read model
        OrderReadModel readModel = new OrderReadModel();
        readModel.setOrderId(event.getOrderId());
        readModel.setUserId(event.getUserId());
        readModel.setUserName(user.getName());           // Denormalized
        readModel.setUserEmail(user.getEmail());         // Denormalized
        readModel.setProductNames(toJson(productNames)); // Denormalized
        readModel.setProductIds(toJson(productIds));     // Denormalized
        readModel.setTotal(event.getTotal());
        readModel.setStatus(event.getStatus());
        readModel.setCreatedAt(event.getCreatedAt());
        
        readRepository.save(readModel);
    }
    
    private String toJson(List<String> list) {
        // Convert to JSON string
        return String.join(",", list);  // Simplified
    }
}
```

**Read Repository:**
```java
@Repository
public interface OrderReadModelRepository extends JpaRepository<OrderReadModel, Long> {
    // Optimized read queries
    List<OrderReadModel> findByUserIdOrderByCreatedAtDesc(Long userId);
    
    List<OrderReadModel> findByStatus(String status);
    
    // Fast queries, no joins needed!
}
```

### Step 6: Query Handler

**GetOrderHistoryQuery:**
```java
public class GetOrderHistoryQuery {
    private Long userId;
    
    // Getters and setters
}
```

**Query Handler:**
```java
@Service
public class OrderQueryHandler {
    
    @Autowired
    private OrderReadModelRepository readRepository;
    
    public List<OrderReadModel> handle(GetOrderHistoryQuery query) {
        // Simple, fast query (no business logic, no joins)
        return readRepository.findByUserIdOrderByCreatedAtDesc(query.getUserId());
    }
    
    public OrderReadModel handle(GetOrderQuery query) {
        return readRepository.findById(query.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException());
    }
}
```

**Query Controller:**
```java
@RestController
@RequestMapping("/api/orders")
public class OrderQueryController {
    
    @Autowired
    private OrderQueryHandler queryHandler;
    
    @GetMapping("/history")
    public ResponseEntity<List<OrderReadModel>> getOrderHistory(
            @RequestParam Long userId) {
        GetOrderHistoryQuery query = new GetOrderHistoryQuery(userId);
        return ResponseEntity.ok(queryHandler.handle(query));
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<OrderReadModel> getOrder(@PathVariable Long id) {
        GetOrderQuery query = new GetOrderQuery(id);
        return ResponseEntity.ok(queryHandler.handle(query));
    }
}
```

### Step 7: Configuration

**application.yml:**
```yaml
spring:
  datasource:
    # Write database
    url: jdbc:postgresql://localhost:5432/order_write_db
    username: order_service
    password: password
  
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect

  # Read database (separate)
  read-datasource:
    url: jdbc:postgresql://localhost:5433/order_read_db
    username: order_service
    password: password

  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: order-read-model-updater
      auto-offset-reset: earliest
```

**Multiple DataSource Configuration:**
```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource")
    public DataSource writeDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties("spring.read-datasource")
    public DataSource readDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean writeEntityManagerFactory() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(writeDataSource());
        em.setPackagesToScan("com.example.order.write");
        em.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
        return em;
    }
    
    @Bean
    public LocalContainerEntityManagerFactoryBean readEntityManagerFactory() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(readDataSource());
        em.setPackagesToScan("com.example.order.read");
        em.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
        return em;
    }
}
```

### Step 8: Docker Compose

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  write-db:
    image: postgres:15
    environment:
      POSTGRES_DB: order_write_db
      POSTGRES_USER: order_service
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"

  read-db:
    image: postgres:15
    environment:
      POSTGRES_DB: order_read_db
      POSTGRES_USER: order_service
      POSTGRES_PASSWORD: password
    ports:
      - "5433:5432"

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
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Tradeoffs

**Pros:**
1. **Independent Scaling**: Scale read and write databases separately
2. **Optimized Models**: Write model optimized for writes, read model for reads
3. **Performance**: Fast reads (no joins, pre-aggregated)
4. **Flexibility**: Multiple read models for different views
5. **Reduced Contention**: Reads don't block writes, writes don't block reads

**Cons:**
1. **Complexity**: Two models to maintain, event handling
2. **Eventual Consistency**: Read model might be slightly stale
3. **More Infrastructure**: Need separate databases, event bus
4. **Data Duplication**: Same data in write and read models
5. **Debugging**: Harder to debug (data flows through events)

### Common Pitfalls

**Pitfall 1: Trying to Keep Read Model Synchronously Updated**

❌ **Wrong:**
```java
@Transactional
public Long createOrder(CreateOrderCommand cmd) {
    Order order = writeRepository.save(order);
    
    // Synchronously update read model (blocks write)
    updateReadModel(order);  // Slow, blocks response
    
    return order.getId();
}
```

✅ **Right:**
```java
public Long createOrder(CreateOrderCommand cmd) {
    Order order = writeRepository.save(order);
    
    // Publish event (asynchronous, non-blocking)
    eventPublisher.publish(new OrderCreatedEvent(order));
    
    return order.getId();  // Returns immediately
}
```

**Pitfall 2: Putting Business Logic in Query Handler**

❌ **Wrong:**
```java
public OrderReadModel getOrder(Long id) {
    OrderReadModel order = readRepository.findById(id);
    
    // Business logic in query handler (wrong!)
    if (order.getStatus().equals("CANCELLED")) {
        throw new OrderCancelledException();  // Should be in command handler
    }
    
    return order;
}
```

✅ **Right:**
```java
// Query handler: just read data, no business logic
public OrderReadModel getOrder(Long id) {
    return readRepository.findById(id)
        .orElseThrow(() -> new OrderNotFoundException());
}

// Business logic in command handler
public void cancelOrder(Long id) {
    Order order = writeRepository.findById(id);
    if (order.getStatus().equals("CANCELLED")) {
        throw new OrderAlreadyCancelledException();
    }
    // Cancel logic...
}
```

**Pitfall 3: Not Handling Event Failures**

❌ **Wrong:**
```java
@KafkaListener(topics = "order-events")
public void handle(OrderCreatedEvent event) {
    // If this fails, read model never updates!
    readRepository.save(createReadModel(event));
}
```

✅ **Right:**
```java
@KafkaListener(topics = "order-events")
public void handle(OrderCreatedEvent event) {
    try {
        readRepository.save(createReadModel(event));
    } catch (Exception e) {
        // Log error, retry, or send to dead letter queue
        log.error("Failed to update read model for order {}", event.getOrderId(), e);
        // Retry mechanism or dead letter queue
    }
}
```

**Pitfall 4: Too Much Denormalization**

❌ **Wrong:**
```java
// Denormalizing entire user object (50+ fields)
class OrderReadModel {
    private User fullUserObject;  // Too much!
    // If user updates any field, need to update all orders
}
```

✅ **Right:**
```java
// Denormalize only what you need for queries
class OrderReadModel {
    private String userName;     // Only what's needed
    private String userEmail;    // Only what's needed
    // If user updates phone number, orders don't need it
}
```

### Performance Gotchas

**Gotcha 1: Read Model Update Lag**

If event processing is slow, read model might be minutes behind. Users see stale data.

**Solution:**
- Monitor event processing lag
- Scale event handlers if needed
- For critical reads, read from write model

**Gotcha 2: Event Replay**

If read model gets corrupted, need to replay all events to rebuild it.

**Solution:**
- Keep event log (event sourcing)
- Can replay events to rebuild read model
- Regular backups of read model

**Gotcha 3: Multiple Read Models**

If you have 5 different read models, need to update all 5 when event occurs.

**Solution:**
- Use event sourcing (single source of truth)
- Each read model subscribes to events independently

### Security Considerations

1. **Read Model Access**: Read model might have different security (less sensitive data)
2. **Event Security**: Events might contain sensitive data (encrypt if needed)
3. **Data Privacy**: Don't denormalize PII unless necessary

---

## 8️⃣ When NOT to Use CQRS

### Anti-Patterns and Misuse Cases

**Don't Use When:**

1. **Simple CRUD Application**
   - Overhead of CQRS > benefits
   - Traditional approach is simpler

2. **Strong Consistency Required**
   - If reads must be immediately consistent with writes
   - CQRS introduces eventual consistency

3. **Low Read/Write Ratio**
   - If you have few reads, not worth optimizing
   - CQRS complexity not justified

4. **Small Team, Simple System**
   - Managing two models is overhead
   - Start simple, add CQRS later if needed

### Signs You've Chosen Wrong

**Red Flags:**
- Read model always needs to be immediately consistent (defeats purpose)
- Simple queries don't benefit from denormalization
- Event processing adds more latency than it saves
- Team struggles to maintain two models

### Better Alternatives for Specific Scenarios

**Scenario 1: Need Immediate Consistency**
- **Alternative**: Read from write model for that query
- **Or**: Use synchronous read model update (defeats some benefits)

**Scenario 2: Simple Application**
- **Alternative**: Traditional single model
- **Or**: Start with single model, add CQRS later if needed

**Scenario 3: Complex Reporting**
- **Alternative**: Data warehouse, ETL pipeline
- **Or**: CQRS with specialized read model for reporting

---

## 9️⃣ Comparison with Alternatives

### CQRS vs Traditional Single Model

| Aspect | CQRS | Traditional |
|--------|------|-------------|
| **Read Performance** | Fast (denormalized) | Slower (joins) |
| **Write Performance** | Fast (normalized) | Fast (normalized) |
| **Consistency** | Eventual | Immediate |
| **Complexity** | Higher | Lower |
| **Scaling** | Scale independently | Scale together |
| **Flexibility** | Multiple read models | Single model |

**When to Choose Each:**
- **CQRS**: High read load, different read/write patterns, need to scale independently
- **Traditional**: Simple app, need immediate consistency, low complexity

### CQRS vs Read Replicas

**Read Replicas:**
- Same schema, replicated database
- Faster reads (separate database)
- Still same model (joins still needed)
- Immediate consistency (synchronous replication)

**CQRS:**
- Different schema, denormalized
- Faster reads (no joins)
- Eventual consistency (asynchronous)

**When to Use Each:**
- **Read Replicas**: Need immediate consistency, same queries work
- **CQRS**: Need different query patterns, OK with eventual consistency

### CQRS vs Materialized Views

**Materialized Views:**
- Database feature, pre-computed views
- Updated on schedule or trigger
- Still in same database
- Database-specific feature

**CQRS:**
- Application-level pattern
- Updated via events
- Separate database
- Database-agnostic

**When to Use Each:**
- **Materialized Views**: Simple denormalization, database supports it
- **CQRS**: Complex logic, need application control, multiple read models

---

## 🔟 Interview Follow-up Questions WITH Answers

### Question 1: How do you handle eventual consistency in CQRS?

**Answer:**

**Acknowledge the Trade-off:**
- CQRS trades immediate consistency for performance and scalability
- Read model is updated asynchronously, so there's a delay

**Strategies:**

**1. Accept Eventual Consistency (Most Common)**
- Most reads don't need immediate consistency
- User can refresh if data seems stale
- Example: Order history page (OK if new order appears after 1 second)

**2. Read from Write Model for Critical Queries**
- For queries that need immediate consistency, read from write model
- Example: "Get order I just created" → read from write model
- Example: "Get order history" → read from read model (eventual consistency OK)

**3. Synchronous Read Model Update (Rare)**
- Update read model synchronously (blocks write)
- Defeats some benefits of CQRS
- Only for critical cases

**4. Optimistic UI Updates**
- Update UI optimistically (assume success)
- If read model doesn't update, show error
- Example: User creates order → UI shows order immediately → If event fails, show error

**5. Version/Timestamp Checking**
- Read model has version/timestamp
- Client can check if data is fresh enough
- If stale, refresh from write model

**Key Insight:** Most applications don't need immediate consistency for all reads. Choose based on business requirements.

---

### Question 2: What happens if the event fails to update the read model?

**Answer:**

**Problem:** Event published, but read model update fails. Read model is stale.

**Solutions:**

**1. Retry Mechanism**
- Event handler retries on failure
- Exponential backoff to avoid overwhelming system
- Example: Retry 3 times with 1s, 2s, 4s delays

**2. Dead Letter Queue (DLQ)**
- Failed events go to DLQ
- Manual or automated processing later
- Can replay events to fix read model

**3. Event Sourcing**
- Keep all events in event store
- Can replay events to rebuild read model
- If read model corrupted, replay from event store

**4. Monitoring and Alerts**
- Monitor event processing lag
- Alert if read model falls behind
- Manual intervention if needed

**5. Compensating Actions**
- If read model update fails, mark data as "stale"
- Show warning to users
- Background job to fix stale data

**6. Idempotent Event Handlers**
- Event handlers should be idempotent
- Can safely replay events
- Example: "UPSERT" instead of "INSERT"

**Example:**
```java
@KafkaListener(topics = "order-events")
public void handle(OrderCreatedEvent event) {
    try {
        // Idempotent: upsert instead of insert
        OrderReadModel existing = readRepository.findById(event.getOrderId());
        if (existing == null) {
            readRepository.save(createReadModel(event));
        } else {
            updateReadModel(existing, event);
        }
    } catch (Exception e) {
        // Send to DLQ for retry
        deadLetterQueue.send(event, e);
        log.error("Failed to update read model", e);
    }
}
```

---

### Question 3: How do you handle updates and deletes in CQRS?

**Answer:**

**Updates:**

**Write Side:**
```java
// Update in write model
public void updateOrderStatus(Long orderId, String status) {
    Order order = writeRepository.findById(orderId);
    order.setStatus(status);
    writeRepository.save(order);
    
    // Publish event
    eventPublisher.publish(new OrderStatusUpdatedEvent(orderId, status));
}
```

**Read Side:**
```java
// Event handler updates read model
@EventListener
public void handle(OrderStatusUpdatedEvent event) {
    OrderReadModel readModel = readRepository.findById(event.getOrderId());
    readModel.setStatus(event.getStatus());
    readRepository.save(readModel);
}
```

**Deletes:**

**Option 1: Soft Delete**
- Mark as deleted in write model
- Publish "OrderDeleted" event
- Read model marks as deleted (still in database, filtered in queries)

**Option 2: Hard Delete**
- Delete from write model
- Publish "OrderDeleted" event
- Read model deletes record

**Option 3: Archive**
- Move to archive table in write model
- Read model removes from main table, keeps in archive

**Example:**
```java
// Soft delete
public void deleteOrder(Long orderId) {
    Order order = writeRepository.findById(orderId);
    order.setDeleted(true);
    writeRepository.save(order);
    
    eventPublisher.publish(new OrderDeletedEvent(orderId));
}

// Read model
@EventListener
public void handle(OrderDeletedEvent event) {
    OrderReadModel readModel = readRepository.findById(event.getOrderId());
    readModel.setDeleted(true);
    readRepository.save(readModel);
}

// Query filters deleted
public List<OrderReadModel> getOrders(Long userId) {
    return readRepository.findByUserIdAndDeletedFalse(userId);
}
```

**Key Insight:** Updates and deletes follow same pattern: update write model → publish event → update read model.

---

### Question 4: Can you have multiple read models in CQRS?

**Answer:**

**Yes! This is a key benefit of CQRS.**

**Different Read Models for Different Views:**

**Example: Order System**

**Read Model 1: Order History (List View)**
```sql
CREATE TABLE order_history_read_model (
    order_id BIGINT,
    user_id BIGINT,
    total DECIMAL,
    status VARCHAR,
    created_at TIMESTAMP,
    -- Minimal data for list view
);
```

**Read Model 2: Order Details (Detail View)**
```sql
CREATE TABLE order_details_read_model (
    order_id BIGINT,
    user_name VARCHAR,
    user_email VARCHAR,
    product_names JSON,
    shipping_address TEXT,
    payment_method VARCHAR,
    -- Full data for detail view
);
```

**Read Model 3: Order Analytics (Reporting)**
```sql
CREATE TABLE order_analytics_read_model (
    order_id BIGINT,
    user_id BIGINT,
    product_category VARCHAR,
    revenue DECIMAL,
    month YEAR_MONTH,
    region VARCHAR,
    -- Aggregated data for analytics
);
```

**Event Handler Updates All:**
```java
@EventListener
public void handle(OrderCreatedEvent event) {
    // Update all read models
    updateOrderHistoryReadModel(event);
    updateOrderDetailsReadModel(event);
    updateOrderAnalyticsReadModel(event);
}
```

**Benefits:**
- Each read model optimized for its use case
- Can use different databases (PostgreSQL for details, ClickHouse for analytics)
- Can scale independently

**Trade-off:** More complexity (more models to maintain)

---

### Question 5: How does CQRS relate to Event Sourcing?

**Answer:**

**They're often used together but are separate patterns:**

**CQRS:**
- Separates read and write models
- Write model can be traditional (current state)
- Read model is denormalized

**Event Sourcing:**
- Store all changes as events
- Current state = replay all events
- Event store is source of truth

**CQRS + Event Sourcing:**
- Write side: Events stored in event store
- Read side: Read models built from events
- Perfect fit: Events update read models

**Architecture:**
```
Command → Command Handler → Event Store (events)
                                    ↓
                            Event Handlers
                                    ↓
                        Read Models (built from events)
```

**Example:**
```java
// Write: Store event
eventStore.append(new OrderCreatedEvent(...));

// Read: Build read model from events
List<Event> events = eventStore.getEventsForOrder(orderId);
OrderReadModel readModel = buildReadModelFromEvents(events);
```

**Benefits of Combining:**
- Event store is source of truth
- Can rebuild read models by replaying events
- Can create new read models from historical events
- Full audit trail

**CQRS Without Event Sourcing:**
- Write model is source of truth (traditional database)
- Events are just for updating read models
- Simpler, but can't rebuild read models easily

**When to Use Each:**
- **CQRS Only**: Simple case, don't need event sourcing benefits
- **CQRS + Event Sourcing**: Need audit trail, want to rebuild read models, complex domain

---

### Question 6: How do you test CQRS systems?

**Answer:**

**Testing Challenges:**
- Two models to test
- Event handling (asynchronous)
- Eventual consistency

**Testing Strategies:**

**1. Unit Tests**

**Command Handler:**
```java
@Test
public void testCreateOrder() {
    CreateOrderCommand cmd = new CreateOrderCommand(...);
    Long orderId = commandHandler.handle(cmd);
    
    // Verify write model
    Order order = writeRepository.findById(orderId);
    assertNotNull(order);
    assertEquals("PENDING", order.getStatus());
    
    // Verify event published (mock)
    verify(eventPublisher).publish(any(OrderCreatedEvent.class));
}
```

**Query Handler:**
```java
@Test
public void testGetOrderHistory() {
    // Setup read model
    OrderReadModel readModel = new OrderReadModel(...);
    readRepository.save(readModel);
    
    // Test query
    List<OrderReadModel> result = queryHandler.handle(
        new GetOrderHistoryQuery(userId)
    );
    
    assertEquals(1, result.size());
}
```

**2. Integration Tests**

**Test Event Flow:**
```java
@Test
public void testEventUpdatesReadModel() {
    // Create order (write side)
    Long orderId = commandHandler.handle(createOrderCommand());
    
    // Wait for event processing
    await().atMost(5, SECONDS).until(() -> {
        return readRepository.findById(orderId).isPresent();
    });
    
    // Verify read model
    OrderReadModel readModel = readRepository.findById(orderId).get();
    assertNotNull(readModel);
    assertEquals("John Doe", readModel.getUserName());
}
```

**3. End-to-End Tests**

**Test Full Flow:**
```java
@Test
public void testCreateOrderAndQuery() {
    // Create order
    Long orderId = restTemplate.postForObject(
        "/api/orders",
        createOrderRequest(),
        Long.class
    );
    
    // Wait for read model update
    Thread.sleep(1000);
    
    // Query order history
    List<OrderReadModel> orders = restTemplate.getForObject(
        "/api/orders/history?userId=123",
        List.class
    );
    
    assertTrue(orders.size() > 0);
}
```

**4. Contract Tests**

**Test Event Contract:**
```java
@Test
public void testOrderCreatedEventContract() {
    OrderCreatedEvent event = new OrderCreatedEvent(...);
    
    // Verify event can be serialized/deserialized
    String json = objectMapper.writeValueAsString(event);
    OrderCreatedEvent deserialized = objectMapper.readValue(
        json,
        OrderCreatedEvent.class
    );
    
    assertEquals(event.getOrderId(), deserialized.getOrderId());
}
```

**Key Testing Principles:**
- Test write and read sides separately
- Test event flow (write → event → read)
- Handle eventual consistency in tests (wait, retry)
- Mock external services (user service, product service)

---

### Question 7: How do you handle schema changes in CQRS?

**Answer:**

**Challenge:** Two models to update (write and read), plus events.

**Strategies:**

**1. Versioned Events**

**Old Event:**
```java
public class OrderCreatedEvent {
    private Long orderId;
    private Long userId;
    private BigDecimal total;
}
```

**New Event (Versioned):**
```java
public class OrderCreatedEventV2 {
    private Long orderId;
    private Long userId;
    private BigDecimal total;
    private String currency;  // New field
    private Integer version = 2;
}
```

**Event Handler Handles Both:**
```java
@EventListener
public void handle(Object event) {
    if (event instanceof OrderCreatedEventV2) {
        handleV2((OrderCreatedEventV2) event);
    } else if (event instanceof OrderCreatedEvent) {
        handleV1((OrderCreatedEvent) event);
    }
}
```

**2. Backward Compatible Changes**

**Add Optional Fields:**
```java
public class OrderCreatedEvent {
    private Long orderId;
    private Long userId;
    private BigDecimal total;
    private String currency;  // New, optional (can be null)
}
```

**Event Handler:**
```java
@EventListener
public void handle(OrderCreatedEvent event) {
    OrderReadModel readModel = new OrderReadModel();
    readModel.setOrderId(event.getOrderId());
    readModel.setCurrency(
        event.getCurrency() != null ? event.getCurrency() : "USD"
    );
    // Old events work (currency is null, defaults to USD)
}
```

**3. Read Model Migration**

**Add Column to Read Model:**
```sql
ALTER TABLE order_read_model
ADD COLUMN currency VARCHAR(3) DEFAULT 'USD';
```

**Update Event Handler:**
```java
@EventListener
public void handle(OrderCreatedEvent event) {
    OrderReadModel readModel = readRepository.findById(event.getOrderId());
    readModel.setCurrency(event.getCurrency());  // New field
    readRepository.save(readModel);
}
```

**4. Event Replay**

**If Read Model Schema Changes:**
- Replay events to rebuild read model
- Event store has all historical events
- Can rebuild read model with new schema

**Example:**
```java
// Rebuild read model from events
public void rebuildReadModel() {
    readRepository.deleteAll();
    
    List<Event> allEvents = eventStore.getAllEvents();
    for (Event event : allEvents) {
        eventHandler.handle(event);  // Process with new schema
    }
}
```

**Key Principles:**
- Keep events backward compatible when possible
- Version events for breaking changes
- Can replay events to update read models
- Test migration paths

---

## 1️⃣1️⃣ One Clean Mental Summary

CQRS separates the model for writing data from the model for reading data. The write model is normalized and optimized for writes (fast inserts, business rules). The read model is denormalized and optimized for reads (fast queries, no joins). When a write happens, an event is published. An event handler asynchronously updates the read model. This allows scaling reads and writes independently and optimizing each for its purpose. The trade-off is eventual consistency: the read model might be slightly behind the write model. Think of it like a restaurant: the chef's notebook (write model) is simple and fast to update, while the printed menu (read model) is beautiful and optimized for customers to read, and it gets updated whenever the chef changes the notebook.

