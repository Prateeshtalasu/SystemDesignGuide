# 🌐 What is a Distributed System?

---

## 0️⃣ Prerequisites

Before diving into distributed systems, you need to understand:

- **Computer/Server**: A machine that runs programs and stores data. Think of it as a single worker who can do tasks.
- **Network**: The communication pathway (like internet cables, WiFi) that allows computers to talk to each other.
- **Process**: A running program on a computer.

If you understand that a computer runs programs and computers can talk to each other over a network, you're ready.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Point

Imagine you built a simple online store. It runs on one computer (server) in your office.

**Day 1**: 10 customers visit. Everything works fine.
**Day 30**: 1,000 customers visit. Server slows down.
**Day 90**: 100,000 customers try to visit. Server crashes.

**The fundamental problem**: A single computer has limits.

- **CPU limit**: Can only process so many calculations per second
- **Memory limit**: Can only hold so much data in RAM
- **Storage limit**: Hard drives have finite space
- **Network limit**: One network card can only handle so many connections

### What Systems Looked Like Before

In the early days, all applications ran on **one big computer** (called a **monolith**):

```
┌─────────────────────────────────┐
│         SINGLE SERVER           │
│  ┌───────────────────────────┐  │
│  │   Your Entire Application │  │
│  │   - User Interface        │  │
│  │   - Business Logic        │  │
│  │   - Database              │  │
│  │   - File Storage          │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### What Breaks Without Distribution?

1. **Single Point of Failure**: If that one server dies, your entire business stops.

   - Real example: In 2017, a single Amazon S3 outage took down thousands of websites because many relied on one region.

2. **Cannot Scale**: You can only make one computer so powerful.

   - You can add more RAM, faster CPU, but there's a ceiling.
   - A single server cannot handle Netflix's 200+ million users streaming simultaneously.

3. **Geographic Latency**: If your server is in New York, users in Tokyo experience slow responses.

   - Light takes ~67ms to travel from New York to Tokyo. Physics cannot be optimized.

4. **No Fault Isolation**: A bug in one part crashes everything.
   - A memory leak in your payment code takes down your entire website.

### Real Examples of the Problem

**Twitter (2008-2010)**: The infamous "Fail Whale" appeared constantly because Twitter ran on a monolithic Ruby on Rails app that couldn't handle growth.

**Early Instagram**: Before Facebook acquisition, Instagram ran on a single PostgreSQL database. They had to frantically scale when they hit millions of users.

---

## 2️⃣ Intuition and Mental Model

### The Restaurant Analogy

Think of a **single server system** as a **one-person restaurant**:

```
┌─────────────────────────────────────────┐
│           ONE-PERSON RESTAURANT         │
│                                         │
│    Chef = Takes orders                  │
│    Chef = Cooks food                    │
│    Chef = Serves tables                 │
│    Chef = Washes dishes                 │
│    Chef = Handles payments              │
│                                         │
│    If chef gets sick → Restaurant CLOSED│
└─────────────────────────────────────────┘
```

Now think of a **distributed system** as a **professional restaurant**:

```
┌─────────────────────────────────────────────────────────┐
│              PROFESSIONAL RESTAURANT                     │
│                                                          │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │
│   │ Waiter  │  │  Chef   │  │ Cashier │  │ Cleaner │   │
│   │ (takes  │  │ (cooks) │  │ (bills) │  │ (dishes)│   │
│   │ orders) │  │         │  │         │  │         │   │
│   └─────────┘  └─────────┘  └─────────┘  └─────────┘   │
│                                                          │
│   If one waiter is sick → Other waiters cover           │
│   Busy night? → Add more waiters                        │
│   Chef overloaded? → Add another chef                   │
└─────────────────────────────────────────────────────────┘
```

**Key insight**: Work is divided among specialists who communicate with each other.

This analogy will be referenced throughout. Remember:

- Each worker = A server/service
- Communication between workers = Network calls
- Kitchen orders = Messages/Requests
- Manager coordinating = Orchestration layer

---

## 3️⃣ How It Works Internally

### Definition from First Principles

A **distributed system** is a collection of independent computers that appears to users as a single coherent system.

Let's break this down word by word:

| Term                       | Meaning                              |
| -------------------------- | ------------------------------------ |
| **Collection**             | Multiple (2 or more)                 |
| **Independent**            | Each can fail without others failing |
| **Computers**              | Physical or virtual machines         |
| **Appears to users**       | Users don't see the complexity       |
| **Single coherent system** | Behaves as one unified application   |

### The Three Defining Characteristics

**1. Multiple Computers Working Together**

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Server A │     │ Server B │     │ Server C │
│ (Orders) │────▶│ (Payment)│────▶│(Shipping)│
└──────────┘     └──────────┘     └──────────┘
```

**2. No Shared Memory**

Unlike threads in a single program that share memory, distributed computers must communicate over the network.

```
Single Computer (Shared Memory):
┌─────────────────────────────┐
│  Thread 1    Thread 2       │
│     │           │           │
│     └─────┬─────┘           │
│           ▼                 │
│    [Shared Memory]          │
└─────────────────────────────┘

Distributed System (Network Communication):
┌───────────┐         ┌───────────┐
│ Computer A│◄──────►│ Computer B│
│ [Memory A]│ Network │ [Memory B]│
└───────────┘         └───────────┘
```

**3. No Global Clock**

Each computer has its own clock. They can drift apart. This creates ordering problems.

```
Computer A's clock: 10:00:00.000
Computer B's clock: 10:00:00.150  ← 150ms ahead!

If A sends message at "10:00:00.100" (A's time)
B receives it at "10:00:00.050" (B's time)
B thinks: "I received a message from the FUTURE?!"
```

### Difference from Monolithic

| Aspect               | Monolithic                   | Distributed                  |
| -------------------- | ---------------------------- | ---------------------------- |
| **Deployment**       | One unit                     | Multiple units               |
| **Scaling**          | Scale entire app             | Scale specific parts         |
| **Failure**          | All or nothing               | Partial failures possible    |
| **Communication**    | Function calls (nanoseconds) | Network calls (milliseconds) |
| **Data Consistency** | Easy (one database)          | Hard (multiple databases)    |
| **Complexity**       | Lower                        | Higher                       |

### Why Distribution is Hard

Peter Deutsch and others at Sun Microsystems identified the **8 Fallacies of Distributed Computing**. These are assumptions that developers make that turn out to be false:

1. **The network is reliable** → Networks fail. Packets get lost.
2. **Latency is zero** → Every network call takes time.
3. **Bandwidth is infinite** → There's a limit to data transfer.
4. **The network is secure** → Anyone can intercept traffic.
5. **Topology doesn't change** → Servers come and go.
6. **There is one administrator** → Multiple teams manage different parts.
7. **Transport cost is zero** → Data transfer costs money.
8. **The network is homogeneous** → Different hardware, protocols exist.

**Real consequence of ignoring these**:

```java
// WRONG: Assumes network always works
public User getUser(String userId) {
    return remoteService.fetchUser(userId);  // What if this times out?
}

// RIGHT: Handles network reality
public User getUser(String userId) {
    try {
        return remoteService.fetchUser(userId);
    } catch (TimeoutException e) {
        return getCachedUser(userId);  // Fallback
    } catch (NetworkException e) {
        throw new ServiceUnavailableException("User service down");
    }
}
```

---

## 4️⃣ Simulation-First Explanation

Let's trace what happens when you open amazon.com and buy a book.

### The Simplest Distributed System

```
┌────────┐         ┌────────────────┐         ┌──────────┐
│  You   │────────▶│  Web Server    │────────▶│ Database │
│(Browser)│◀────────│  (Application) │◀────────│          │
└────────┘         └────────────────┘         └──────────┘
```

Even this simple setup is distributed: 3 different computers communicating over a network.

### Step-by-Step: Buying a Book

**Step 1: You click "Buy Now"**

Your browser sends an HTTP request:

```http
POST /api/orders HTTP/1.1
Host: amazon.com
Content-Type: application/json

{
  "userId": "12345",
  "bookId": "978-0134685991",
  "quantity": 1
}
```

**Step 2: Request hits Load Balancer**

Amazon has thousands of servers. A load balancer decides which one handles your request.

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    │  (picks server) │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
     ┌──────────┐      ┌──────────┐      ┌──────────┐
     │ Server 1 │      │ Server 2 │      │ Server 3 │
     │  (busy)  │      │  (free)  │      │  (busy)  │
     └──────────┘      └──────────┘      └──────────┘
                             │
                    Your request goes here
```

**Step 3: Order Service processes request**

```
┌─────────────────────────────────────────────────────────────┐
│                     ORDER SERVICE                            │
│                                                              │
│  1. Validate user exists      → calls User Service          │
│  2. Check book in stock       → calls Inventory Service     │
│  3. Calculate price           → calls Pricing Service       │
│  4. Process payment           → calls Payment Service       │
│  5. Create order record       → writes to Order Database    │
│  6. Trigger shipping          → sends message to Queue      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Step 4: Services communicate**

```
Order Service                    Inventory Service
     │                                  │
     │  POST /inventory/check           │
     │  {"bookId": "978-0134685991"}    │
     │─────────────────────────────────▶│
     │                                  │
     │  {"available": true, "qty": 47}  │
     │◀─────────────────────────────────│
     │                                  │
```

**Step 5: Payment processed**

```
Order Service                    Payment Service                 Bank API
     │                                  │                            │
     │  POST /payment/charge            │                            │
     │  {"userId": "12345",             │                            │
     │   "amount": 49.99}               │                            │
     │─────────────────────────────────▶│                            │
     │                                  │  External API call         │
     │                                  │───────────────────────────▶│
     │                                  │                            │
     │                                  │  {"approved": true}        │
     │                                  │◀───────────────────────────│
     │  {"transactionId": "TXN123"}     │                            │
     │◀─────────────────────────────────│                            │
```

**Step 6: Async shipping notification**

```
Order Service                    Message Queue                 Shipping Service
     │                                  │                            │
     │  PUBLISH to "orders" topic       │                            │
     │  {"orderId": "ORD456",           │                            │
     │   "address": "123 Main St"}      │                            │
     │─────────────────────────────────▶│                            │
     │                                  │                            │
     │  (Order Service continues,       │  SUBSCRIBE to "orders"     │
     │   doesn't wait)                  │◀───────────────────────────│
     │                                  │                            │
     │                                  │  Deliver message           │
     │                                  │───────────────────────────▶│
     │                                  │                            │
     │                                  │         Process shipping   │
```

### What the Bytes Actually Look Like

When Order Service calls Inventory Service, here's the actual network traffic:

```
TCP Packet (simplified):
┌──────────────────────────────────────────────────────────────┐
│ Source IP: 10.0.1.15 (Order Service)                         │
│ Dest IP: 10.0.2.23 (Inventory Service)                       │
│ Source Port: 52431                                           │
│ Dest Port: 8080                                              │
├──────────────────────────────────────────────────────────────┤
│ HTTP Payload:                                                │
│ POST /inventory/check HTTP/1.1                               │
│ Host: inventory-service.internal                             │
│ Content-Type: application/json                               │
│ X-Request-ID: abc-123-def                                    │
│                                                              │
│ {"bookId":"978-0134685991","warehouseId":"WH-EAST-1"}       │
└──────────────────────────────────────────────────────────────┘
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Real Systems at Real Companies

**Netflix Architecture**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        NETFLIX ARCHITECTURE                          │
│                                                                      │
│  ┌─────────┐    ┌─────────────┐    ┌──────────────────────────────┐ │
│  │  User   │───▶│   Zuul      │───▶│  Microservices (700+)        │ │
│  │  App    │    │  (Gateway)  │    │  - User Service              │ │
│  └─────────┘    └─────────────┘    │  - Recommendation Service    │ │
│                                     │  - Streaming Service         │ │
│                                     │  - Billing Service           │ │
│                                     └──────────────────────────────┘ │
│                                                   │                  │
│                                     ┌─────────────┴────────────┐    │
│                                     │    Data Stores           │    │
│                                     │  - Cassandra (user data) │    │
│                                     │  - EVCache (caching)     │    │
│                                     │  - Kafka (events)        │    │
│                                     └──────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

Netflix has 700+ microservices handling 200+ million subscribers.

**Uber Architecture**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         UBER ARCHITECTURE                            │
│                                                                      │
│   Driver App ───┐                     ┌─── Matching Service         │
│                 │    ┌───────────┐    │                             │
│   Rider App  ───┼───▶│  Gateway  │────┼─── Pricing Service          │
│                 │    └───────────┘    │                             │
│   Web App   ────┘                     ├─── Maps Service             │
│                                       │                             │
│                                       ├─── Payment Service          │
│                                       │                             │
│                                       └─── Notification Service     │
│                                                                      │
│   All services communicate via:                                      │
│   - gRPC (synchronous)                                              │
│   - Kafka (asynchronous events)                                     │
│   - Redis (caching)                                                 │
└─────────────────────────────────────────────────────────────────────┘
```

### Real Workflows and Tooling

**Service Discovery**: How services find each other

```yaml
# Netflix Eureka - Service Registration
eureka:
  client:
    serviceUrl:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    hostname: order-service
# When Order Service starts, it registers with Eureka
# Other services can then find it by name, not IP address
```

**Container Orchestration**: How services are deployed

```yaml
# Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3 # Run 3 instances
  selector:
    matchLabels:
      app: order-service
  template:
    spec:
      containers:
        - name: order-service
          image: mycompany/order-service:v1.2.3
          resources:
            limits:
              memory: "512Mi"
              cpu: "500m"
```

### What is Automated vs Manual

| Aspect             | Automated           | Manual                        |
| ------------------ | ------------------- | ----------------------------- |
| Service deployment | CI/CD pipelines     | Initial setup                 |
| Scaling up/down    | Auto-scaling rules  | Capacity planning             |
| Health checks      | Kubernetes probes   | Defining what "healthy" means |
| Load balancing     | Ingress controllers | Choosing strategy             |
| Log aggregation    | Fluentd/Logstash    | Alert thresholds              |

### Production War Stories

**Amazon's 2017 S3 Outage**:

- An engineer ran a command to remove a small number of servers
- A typo caused more servers than intended to be removed
- This cascaded and took down S3 in US-East-1
- Thousands of websites went down
- **Lesson**: Distributed systems need safeguards against human error

**GitHub's 2018 Database Incident**:

- Network partition between data centers
- Database failover triggered
- Old primary came back online, causing split-brain
- **Lesson**: Network partitions are real; CAP theorem matters

---

## 6️⃣ How to Implement a Basic Distributed System

Let's build a simple distributed system with two services.

### Project Structure

```
distributed-demo/
├── order-service/
│   ├── pom.xml
│   └── src/main/java/com/example/order/
│       ├── OrderServiceApplication.java
│       ├── controller/OrderController.java
│       └── client/InventoryClient.java
├── inventory-service/
│   ├── pom.xml
│   └── src/main/java/com/example/inventory/
│       ├── InventoryServiceApplication.java
│       └── controller/InventoryController.java
└── docker-compose.yml
```

### Maven Dependencies (pom.xml for Order Service)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>order-service</artifactId>
    <version>1.0.0</version>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- Web framework for REST APIs -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- For making HTTP calls to other services -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
        </dependency>

        <!-- Health checks and metrics -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
    </dependencies>
</project>
```

### Inventory Service (The service being called)

```java
// InventoryServiceApplication.java
package com.example.inventory;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication  // Marks this as a Spring Boot application
public class InventoryServiceApplication {
    public static void main(String[] args) {
        // Starts the embedded web server and Spring context
        SpringApplication.run(InventoryServiceApplication.class, args);
    }
}
```

```java
// InventoryController.java
package com.example.inventory.controller;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@RestController  // Marks this class as a REST API controller
@RequestMapping("/api/inventory")  // Base path for all endpoints
public class InventoryController {

    // Simulated database - in production, this would be a real database
    // ConcurrentHashMap is thread-safe for multiple simultaneous requests
    private final Map<String, Integer> inventory = new ConcurrentHashMap<>();

    // Constructor - initialize with some test data
    public InventoryController() {
        inventory.put("BOOK-001", 100);
        inventory.put("BOOK-002", 50);
        inventory.put("BOOK-003", 0);  // Out of stock
    }

    /**
     * Check if a product is available
     *
     * @param productId The product to check
     * @param quantity How many units needed
     * @return JSON with availability status
     */
    @GetMapping("/check/{productId}")
    public ResponseEntity<InventoryResponse> checkInventory(
            @PathVariable String productId,
            @RequestParam(defaultValue = "1") int quantity) {

        // Look up current stock
        Integer currentStock = inventory.get(productId);

        // Product doesn't exist
        if (currentStock == null) {
            return ResponseEntity.ok(new InventoryResponse(
                productId,
                false,
                0,
                "Product not found"
            ));
        }

        // Check if enough stock
        boolean available = currentStock >= quantity;

        return ResponseEntity.ok(new InventoryResponse(
            productId,
            available,
            currentStock,
            available ? "In stock" : "Insufficient stock"
        ));
    }

    /**
     * Reserve inventory (reduce stock)
     * Called when an order is placed
     */
    @PostMapping("/reserve/{productId}")
    public ResponseEntity<InventoryResponse> reserveInventory(
            @PathVariable String productId,
            @RequestParam int quantity) {

        // Atomic operation to prevent race conditions
        Integer currentStock = inventory.get(productId);

        if (currentStock == null || currentStock < quantity) {
            return ResponseEntity.badRequest().body(new InventoryResponse(
                productId, false, 0, "Cannot reserve - insufficient stock"
            ));
        }

        // Reduce stock
        inventory.put(productId, currentStock - quantity);

        return ResponseEntity.ok(new InventoryResponse(
            productId,
            true,
            currentStock - quantity,
            "Reserved successfully"
        ));
    }

    // Response record (Java 17+ feature)
    public record InventoryResponse(
        String productId,
        boolean available,
        int currentStock,
        String message
    ) {}
}
```

### Order Service (The service making calls)

```java
// OrderServiceApplication.java
package com.example.order;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.web.reactive.function.client.WebClient;

@SpringBootApplication
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }

    /**
     * WebClient is Spring's modern HTTP client for making REST calls
     * It's non-blocking and supports reactive programming
     */
    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .baseUrl("http://localhost:8081")  // Inventory service URL
            .build();
    }
}
```

```java
// InventoryClient.java
package com.example.order.client;

import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;
import org.springframework.web.reactive.function.client.WebClientResponseException;
import java.time.Duration;

@Component  // Spring will manage this class as a bean
public class InventoryClient {

    private final WebClient webClient;

    // Constructor injection - Spring provides the WebClient
    public InventoryClient(WebClient webClient) {
        this.webClient = webClient;
    }

    /**
     * Check inventory availability
     *
     * This method demonstrates proper distributed system practices:
     * 1. Timeout handling - don't wait forever
     * 2. Error handling - network calls can fail
     * 3. Fallback behavior - what to do when service is down
     */
    public InventoryResponse checkInventory(String productId, int quantity) {
        try {
            return webClient
                .get()
                .uri("/api/inventory/check/{productId}?quantity={qty}",
                     productId, quantity)
                .retrieve()
                .bodyToMono(InventoryResponse.class)
                .timeout(Duration.ofSeconds(5))  // Don't wait more than 5 seconds
                .block();  // Convert async to sync for simplicity

        } catch (WebClientResponseException e) {
            // HTTP error from inventory service (4xx, 5xx)
            System.err.println("Inventory service error: " + e.getStatusCode());
            return new InventoryResponse(productId, false, 0,
                "Inventory service error");

        } catch (Exception e) {
            // Network timeout, connection refused, etc.
            System.err.println("Failed to reach inventory service: " + e.getMessage());
            return new InventoryResponse(productId, false, 0,
                "Inventory service unavailable");
        }
    }

    public record InventoryResponse(
        String productId,
        boolean available,
        int currentStock,
        String message
    ) {}
}
```

```java
// OrderController.java
package com.example.order.controller;

import com.example.order.client.InventoryClient;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.UUID;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final InventoryClient inventoryClient;

    public OrderController(InventoryClient inventoryClient) {
        this.inventoryClient = inventoryClient;
    }

    /**
     * Create a new order
     *
     * This demonstrates the distributed nature:
     * 1. Receive request from user
     * 2. Call inventory service to check/reserve stock
     * 3. Create order only if inventory available
     */
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@RequestBody OrderRequest request) {

        // Step 1: Check inventory (calls another service!)
        var inventoryCheck = inventoryClient.checkInventory(
            request.productId(),
            request.quantity()
        );

        // Step 2: If not available, reject order
        if (!inventoryCheck.available()) {
            return ResponseEntity.badRequest().body(new OrderResponse(
                null,
                "REJECTED",
                "Product not available: " + inventoryCheck.message()
            ));
        }

        // Step 3: Create order (in real system, would save to database)
        String orderId = UUID.randomUUID().toString();

        // In production: would also call inventory to RESERVE stock
        // and payment service to charge customer

        return ResponseEntity.ok(new OrderResponse(
            orderId,
            "CREATED",
            "Order created successfully"
        ));
    }

    // Request/Response records
    public record OrderRequest(String productId, int quantity, String customerId) {}
    public record OrderResponse(String orderId, String status, String message) {}
}
```

### Configuration Files

```yaml
# order-service/src/main/resources/application.yml
server:
  port: 8080 # Order service runs on 8080

spring:
  application:
    name: order-service # Used for service discovery and logging

# Actuator endpoints for health checks
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics
```

```yaml
# inventory-service/src/main/resources/application.yml
server:
  port: 8081 # Inventory service runs on 8081

spring:
  application:
    name: inventory-service

management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics
```

### Docker Compose (Running Both Services)

```yaml
# docker-compose.yml
version: "3.8"

services:
  inventory-service:
    build: ./inventory-service
    ports:
      - "8081:8081"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  order-service:
    build: ./order-service
    ports:
      - "8080:8080"
    depends_on:
      inventory-service:
        condition: service_healthy
    environment:
      - INVENTORY_SERVICE_URL=http://inventory-service:8081
```

### Commands to Run and Verify

```bash
# Terminal 1: Start Inventory Service
cd inventory-service
mvn spring-boot:run

# Terminal 2: Start Order Service
cd order-service
mvn spring-boot:run

# Terminal 3: Test the distributed system

# Check inventory directly
curl http://localhost:8081/api/inventory/check/BOOK-001

# Expected response:
# {"productId":"BOOK-001","available":true,"currentStock":100,"message":"In stock"}

# Create an order (calls inventory service internally)
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"productId":"BOOK-001","quantity":2,"customerId":"CUST-123"}'

# Expected response:
# {"orderId":"abc-123-...","status":"CREATED","message":"Order created successfully"}

# Try ordering out-of-stock item
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"productId":"BOOK-003","quantity":1,"customerId":"CUST-123"}'

# Expected response:
# {"orderId":null,"status":"REJECTED","message":"Product not available: Insufficient stock"}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Why Not Just Use a Monolith?

| Situation                       | Choose Monolith | Choose Distributed |
| ------------------------------- | --------------- | ------------------ |
| Small team (<10 devs)           | ✅              | ❌                 |
| Early startup                   | ✅              | ❌                 |
| Simple domain                   | ✅              | ❌                 |
| Need to scale specific parts    | ❌              | ✅                 |
| Large team (>50 devs)           | ❌              | ✅                 |
| Multiple deployment frequencies | ❌              | ✅                 |

### What Breaks at Scale

**1. Network becomes the bottleneck**

```
Monolith: Function call = ~1 nanosecond
Distributed: Network call = ~1 millisecond (1,000,000x slower!)

If your service makes 100 internal calls per request:
- Monolith: 100 nanoseconds
- Distributed: 100 milliseconds (minimum!)
```

**2. Debugging becomes exponentially harder**

```
Monolith: Stack trace shows exactly what happened
Distributed: Request touched 15 services, which one failed?
             Need distributed tracing (Jaeger, Zipkin)
```

**3. Data consistency is no longer guaranteed**

```
Monolith:
  BEGIN TRANSACTION
    deduct_inventory()
    charge_payment()
    create_order()
  COMMIT  -- All or nothing

Distributed:
  call inventory_service.reserve()  -- Success
  call payment_service.charge()     -- FAILS!
  -- Now what? Inventory is reserved but payment failed!
  -- Need saga pattern or distributed transactions
```

### Common Misconfigurations

**1. No timeouts**

```java
// WRONG: Will hang forever if inventory service is slow
webClient.get().uri("/inventory").retrieve().block();

// RIGHT: Always set timeouts
webClient.get().uri("/inventory")
    .retrieve()
    .timeout(Duration.ofSeconds(5))
    .block();
```

**2. No retry with backoff**

```java
// WRONG: Retry immediately hammers the failing service
for (int i = 0; i < 3; i++) {
    try {
        return callService();
    } catch (Exception e) {
        // Retry immediately
    }
}

// RIGHT: Exponential backoff
int delay = 100;
for (int i = 0; i < 3; i++) {
    try {
        return callService();
    } catch (Exception e) {
        Thread.sleep(delay);
        delay *= 2;  // 100ms, 200ms, 400ms
    }
}
```

**3. Synchronous chains**

```
WRONG: A → B → C → D → E (each waits for the next)
       If E is slow, everything is slow

RIGHT: A → Message Queue → B, C, D, E process independently
       A returns immediately
```

### Performance Gotchas

- **N+1 query problem**: Calling a service in a loop instead of batch
- **No connection pooling**: Creating new connections for each request
- **Large payloads**: Sending entire objects when only IDs are needed
- **No caching**: Calling the same service repeatedly for same data

### Security Considerations

- **Service-to-service authentication**: Services must verify each other
- **Network encryption**: Use TLS for all internal communication
- **Secret management**: Don't hardcode credentials; use Vault or similar
- **Input validation**: Each service must validate input (don't trust other services)

---

## 8️⃣ When NOT to Use Distributed Systems

### Anti-Patterns and Misuse Cases

**1. "Microservices for a TODO app"**

If your app has 3 database tables and 2 developers, you don't need 10 microservices.

**2. "Distributed because it's modern"**

Complexity has a cost. Every network call can fail. Every service needs monitoring.

**3. "One service per database table"**

Services should align with business capabilities, not database structure.

### Situations Where This is Overkill

- Prototype or MVP (just ship it!)
- Team of 1-5 developers
- Application with low traffic (<1000 users)
- Simple CRUD application
- Short-lived project

### Better Alternatives for Specific Scenarios

| Scenario               | Instead of Distributed         | Use                         |
| ---------------------- | ------------------------------ | --------------------------- |
| Need to scale database | Multiple services              | Read replicas               |
| Need async processing  | Message queue between services | Background jobs in monolith |
| Need team independence | Separate services              | Modular monolith            |
| Need fault isolation   | Separate services              | Bulkheads in monolith       |

### Signs You've Chosen the Wrong Tool

- Every feature requires changes to 5+ services
- Debugging takes hours instead of minutes
- Deployment is a multi-day event
- Team spends more time on infrastructure than features
- Simple changes require coordinated releases

---

## 9️⃣ Comparison with Alternatives

### Monolith vs Microservices vs Modular Monolith

```
MONOLITH:
┌──────────────────────────────────────┐
│  Everything in one deployable unit   │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐        │
│  │User│ │Order│ │Pay │ │Ship│        │
│  └────┘ └────┘ └────┘ └────┘        │
│         One Database                 │
└──────────────────────────────────────┘

MICROSERVICES:
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│ User │ │Order │ │ Pay  │ │ Ship │
│ Svc  │ │ Svc  │ │ Svc  │ │ Svc  │
│ [DB] │ │ [DB] │ │ [DB] │ │ [DB] │
└──────┘ └──────┘ └──────┘ └──────┘
   │         │         │        │
   └─────────┴────┬────┴────────┘
                  │
            Network calls

MODULAR MONOLITH:
┌──────────────────────────────────────┐
│  One deployable, clear boundaries    │
│  ┌────────┐ ┌────────┐ ┌────────┐   │
│  │ User   │ │ Order  │ │Payment │   │
│  │ Module │ │ Module │ │ Module │   │
│  └───┬────┘ └───┬────┘ └───┬────┘   │
│      │          │          │         │
│      └──────────┴──────────┘         │
│           Internal APIs              │
│          One Database                │
└──────────────────────────────────────┘
```

### When to Choose Each

| Factor                    | Monolith     | Modular Monolith | Microservices |
| ------------------------- | ------------ | ---------------- | ------------- |
| Team size                 | 1-10         | 10-50            | 50+           |
| Deployment frequency      | Same for all | Same for all     | Independent   |
| Scaling needs             | Uniform      | Uniform          | Per-service   |
| Operational complexity    | Low          | Low              | High          |
| Development speed (early) | Fast         | Fast             | Slow          |
| Development speed (late)  | Slow         | Medium           | Fast          |

### Real-World Examples of Companies Choosing

**Shopify**: Chose modular monolith. 3000+ developers, one Rails app, clear module boundaries.

**Amazon**: Started as monolith, evolved to microservices as they scaled to millions of products.

**Netflix**: Fully microservices. 700+ services. Needed because of global scale and team size.

**Basecamp**: Stayed monolith. Small team, simple product, no need for distribution.

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is a distributed system?**

A: A distributed system is a collection of independent computers that work together and appear to users as a single system. The key characteristics are: (1) multiple machines, (2) no shared memory, so they communicate over network, (3) no global clock, so ordering is tricky, and (4) partial failures are possible, meaning one machine can fail while others continue.

**Q: Why would you choose microservices over a monolith?**

A: I would choose microservices when: (1) the team is large and needs to work independently, (2) different parts of the system have different scaling needs, (3) we need different deployment frequencies for different features, or (4) we need fault isolation so one component's failure doesn't take down everything. However, for small teams or early-stage products, a monolith is usually better because it's simpler to develop, deploy, and debug.

### L5 (Mid-Level) Questions

**Q: How do you handle failures in distributed systems?**

A: I use multiple strategies: (1) Timeouts, so we don't wait forever for a response. (2) Retries with exponential backoff, so we give the service time to recover. (3) Circuit breakers, which stop calling a failing service temporarily. (4) Fallbacks, where we return cached data or degraded functionality. (5) Bulkheads, which isolate failures to prevent cascading. (6) Health checks and monitoring, so we detect issues early.

**Q: How do you ensure data consistency across services?**

A: There are several approaches depending on requirements: (1) For eventual consistency, I use event-driven architecture where services publish events and others react. (2) For operations that must be atomic, I use the Saga pattern with compensating transactions. (3) For critical financial operations, I might use two-phase commit, though it has availability tradeoffs. (4) I also use idempotency keys to handle retries safely. The key is understanding the business requirements, as not everything needs strong consistency.

### L6 (Senior) Questions

**Q: How would you migrate a monolith to microservices?**

A: I would follow the Strangler Fig pattern: (1) First, identify bounded contexts in the monolith and establish clear interfaces. (2) Start with the least critical, most independent service. (3) Put a facade/API gateway in front of the monolith. (4) Extract one service at a time, routing traffic through the gateway. (5) Keep both implementations running in parallel initially. (6) Gradually shift traffic to the new service. (7) Only proceed to the next service after the current one is stable. I would NOT do a big-bang rewrite. The key metrics I'd track are latency, error rates, and developer velocity.

**Q: What are the tradeoffs between synchronous and asynchronous communication in distributed systems?**

A: Synchronous (REST/gRPC): Simpler to understand and debug. Client gets immediate response. But creates tight coupling, and if the downstream service is slow or down, the caller is affected. Good for queries and operations where the user needs immediate feedback.

Asynchronous (message queues): Decouples services. Caller isn't blocked. Better fault tolerance since messages are persisted. But harder to debug, eventual consistency, and more infrastructure to manage. Good for operations that don't need immediate response, like sending emails or processing orders.

In practice, I use both: synchronous for reads and user-facing operations, asynchronous for writes and background processing.

---

## 1️⃣1️⃣ One Clean Mental Summary

A distributed system is like a professional restaurant where specialized workers (servers) collaborate over communication (network) to serve customers. Unlike a one-person restaurant (monolith) that stops when the chef is sick, a distributed restaurant can handle one waiter being absent. The tradeoff is coordination complexity: workers must communicate, orders can get lost, and the kitchen might be overwhelmed. You choose distribution when you need to scale beyond what one person can handle, but you pay the price in operational complexity. Start simple, distribute only when necessary, and always design for failure.
