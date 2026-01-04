# API Gateway Patterns in Microservices

## 0️⃣ Prerequisites

Before diving into this topic, you need to understand:

- **Microservices Architecture**: Service independence and communication (Phase 10, Topic 1)
- **Load Balancers**: How traffic is distributed (Phase 2, Topic 6)
- **Reverse Proxy**: Forward proxy vs reverse proxy concepts (Phase 2, Topic 7)
- **REST APIs**: HTTP methods, status codes, request/response patterns
- **Authentication/Authorization**: JWT tokens, OAuth, API keys
- **Service Discovery**: How services find each other (Phase 10, Topic 4)

**Quick refresher**: In microservices, clients need to call multiple services. Without an API Gateway, clients must know about all services, handle service discovery, manage authentication for each service, and aggregate responses. An API Gateway provides a single entry point that handles routing, authentication, rate limiting, and response aggregation, simplifying client interactions with microservices.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

In a microservices architecture, you have many services that clients need to access:

```
Client needs to call:
  - Order Service (create order)
  - Payment Service (process payment)
  - Inventory Service (check stock)
  - Shipping Service (calculate shipping)
  - Notification Service (send confirmation)
```

**The Problem**: How do clients interact with these services when:
- Clients must know about all services and their endpoints
- Each service has different authentication requirements
- Clients need to make multiple calls and aggregate responses
- Services have different protocols (REST, gRPC, WebSocket)
- Services are behind firewalls and not directly accessible
- Services change frequently, breaking client code

### What Systems Looked Like Before API Gateway

**Direct Service Access (Complexity Explosion):**

```
Mobile App
  ├── Calls Order Service: http://order-service:8080/api/orders
  ├── Calls Payment Service: http://payment-service:8080/api/payments
  ├── Calls Inventory Service: http://inventory-service:8080/api/inventory
  ├── Calls Shipping Service: http://shipping-service:8080/api/shipping
  └── Calls Notification Service: http://notification-service:8080/api/notifications

Problems:
  - Mobile app knows about 5 different services
  - Mobile app handles authentication for each service
  - Mobile app aggregates responses from multiple services
  - Mobile app breaks when service URLs change
  - Mobile app makes 5 network calls (slow, battery drain)
```

**Web Application:**

```
Web App
  ├── Calls Order Service
  ├── Calls Payment Service
  ├── Calls Inventory Service
  ├── Calls Shipping Service
  └── Calls Notification Service

Same problems as mobile app, plus:
  - CORS issues (services on different domains)
  - Security concerns (exposing internal services)
```

### What Breaks Without API Gateway

**At Scale:**

```
10 client types (mobile, web, admin, partner APIs)
× 50 microservices
= 500 integration points to manage
× Different auth, protocols, versions
= Operational nightmare
```

**Real-World Failures:**

- **Service Discovery**: Client doesn't know service IPs. Services scale, IPs change. Client breaks.

- **Multiple Network Calls**: Client needs order details, payment status, shipping info. Makes 3 separate calls. Slow, battery drain, poor UX.

- **Authentication Complexity**: Each service requires different auth. Client manages 5 different auth mechanisms. Complex, error-prone.

- **CORS Issues**: Web app calls services on different domains. CORS errors. Can't access services.

- **Service Changes**: Service changes URL or API. All clients break. Need to update all clients.

- **Security Exposure**: Internal services exposed to internet. Security risk.

### Real Examples of the Problem

**Netflix (Pre-API Gateway)**:
- Multiple client types (web, mobile, TV, partner)
- Hundreds of microservices
- Clients directly called services
- Complex client code, frequent breakages
- Created Zuul API Gateway to solve this

**Amazon (Early Microservices)**:
- Multiple client applications
- Services directly exposed
- Complex client-side logic
- Created API Gateway pattern

**Uber (Multi-Client Architecture)**:
- Rider app, driver app, web app, admin panel
- Each called services directly
- Duplicated logic across clients
- Created API Gateway for unified access

---

## 2️⃣ Intuition and Mental Model

### The Hotel Reception Desk Analogy

Think of **API Gateway** like a hotel reception desk.

**Without API Gateway (Direct Access):**

```
You (Guest) want hotel services:
  - Room service: Call kitchen directly
  - Housekeeping: Call housekeeping directly
  - Concierge: Call concierge directly
  - Billing: Call accounting directly
  - Maintenance: Call maintenance directly

Problems:
  - You need 5 different phone numbers
  - You handle payment for each service
  - You coordinate between services
  - Services change numbers, you're lost
```

**With API Gateway (Reception Desk):**

```
You (Guest) want hotel services:
  - You call reception: "I want room service"
  - Reception routes to kitchen
  - Reception handles payment
  - Reception coordinates services
  - You only know one number: reception

Benefits:
  - Single point of contact
  - Reception handles complexity
  - Services can change, you don't care
  - Reception provides unified experience
```

**API Gateway Components:**

- **Reception Desk**: API Gateway (single entry point)
- **Services**: Hotel departments (kitchen, housekeeping, etc.)
- **Routing**: Reception routes requests to right department
- **Authentication**: Reception verifies you're a guest
- **Aggregation**: Reception coordinates multiple services
- **Protocol Translation**: Reception translates your request to department format

### The Key Mental Model

**API Gateway is like a reception desk for your microservices:**

- **Clients** only know one entry point (reception)
- **Gateway** routes requests to appropriate services (departments)
- **Gateway** handles cross-cutting concerns (auth, rate limiting, logging)
- **Gateway** aggregates responses from multiple services
- **Services** stay hidden from clients (internal departments)

This analogy helps us understand:
- Why single entry point matters (one reception, not many departments)
- Why routing is needed (reception knows where to route)
- Why aggregation helps (reception coordinates services)
- Why it simplifies clients (clients only know reception)

---

## 3️⃣ How It Works Internally

API Gateway follows these patterns:

### Pattern 1: Request Routing

```
┌─────────────────────────────────────────────────────────┐
│              REQUEST ROUTING                            │
└─────────────────────────────────────────────────────────┘

Client
  ↓
API Gateway
  ├── Receives: POST /api/orders
  ├── Authenticates request
  ├── Routes to: Order Service
  └── Returns response
  ↓
Order Service
```

**Flow:**
1. Client sends request to gateway: `POST https://api.example.com/orders`
2. Gateway authenticates client
3. Gateway routes to Order Service: `POST http://order-service:8080/api/orders`
4. Gateway returns response to client

### Pattern 2: Request Aggregation

```
┌─────────────────────────────────────────────────────────┐
│              REQUEST AGGREGATION                        │
└─────────────────────────────────────────────────────────┘

Client
  ↓
API Gateway
  ├── Receives: GET /api/order-details/{orderId}
  ├── Calls Order Service: GET /orders/{orderId}
  ├── Calls Payment Service: GET /payments/order/{orderId}
  ├── Calls Shipping Service: GET /shipping/order/{orderId}
  ├── Aggregates responses
  └── Returns combined response
  ↓
Order Service, Payment Service, Shipping Service
```

**Flow:**
1. Client requests order details: `GET /api/order-details/123`
2. Gateway calls Order Service: `GET /orders/123`
3. Gateway calls Payment Service: `GET /payments/order/123`
4. Gateway calls Shipping Service: `GET /shipping/order/123`
5. Gateway aggregates responses into single response
6. Gateway returns to client

### Pattern 3: Protocol Translation

```
┌─────────────────────────────────────────────────────────┐
│              PROTOCOL TRANSLATION                       │
└─────────────────────────────────────────────────────────┘

Client (REST)
  ↓
API Gateway
  ├── Receives: REST request
  ├── Translates to: gRPC
  ├── Calls: gRPC service
  ├── Translates response: gRPC → REST
  └── Returns: REST response
  ↓
Microservice (gRPC)
```

**Flow:**
1. Client sends REST request
2. Gateway translates to gRPC
3. Gateway calls gRPC service
4. Gateway translates gRPC response to REST
5. Gateway returns REST response

### Pattern 4: Backend for Frontend (BFF)

```
┌─────────────────────────────────────────────────────────┐
│              BACKEND FOR FRONTEND                       │
└─────────────────────────────────────────────────────────┘

Mobile App
  ↓
Mobile BFF (API Gateway)
  ├── Optimized for mobile
  ├── Aggregates responses
  └── Returns mobile-optimized format
  ↓
Services

Web App
  ↓
Web BFF (API Gateway)
  ├── Optimized for web
  ├── Different aggregation
  └── Returns web-optimized format
  ↓
Services
```

**Flow:**
1. Mobile app calls Mobile BFF
2. Mobile BFF aggregates services, returns mobile-optimized response
3. Web app calls Web BFF
4. Web BFF aggregates services, returns web-optimized response

---

## 4️⃣ Simulation-First Explanation

Let's trace a request through API Gateway:

### Scenario: Mobile App Creates Order

**Setup:**
- Mobile App: Client
- API Gateway: Single entry point
- Order Service, Payment Service, Inventory Service: Backend services

**Step 1: Client Sends Request**

```http
POST https://api.example.com/v1/orders
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "customerId": "123",
  "items": [
    {"productId": "456", "quantity": 2},
    {"productId": "789", "quantity": 1}
  ]
}
```

**Step 2: API Gateway Receives Request**

```java
// API Gateway receives request
@RestController
@RequestMapping("/v1")
public class OrderGatewayController {
    
    @Autowired
    private AuthenticationService authService;
    
    @Autowired
    private OrderServiceClient orderService;
    
    @Autowired
    private InventoryServiceClient inventoryService;
    
    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> createOrder(
            @RequestHeader("Authorization") String token,
            @RequestBody CreateOrderRequest request) {
        
        // 1. Authenticate
        User user = authService.authenticate(token);
        if (user == null) {
            return ResponseEntity.status(401).build();
        }
        
        // 2. Validate request
        if (!validateRequest(request)) {
            return ResponseEntity.status(400).build();
        }
        
        // 3. Check inventory (aggregation)
        InventoryStatus inventory = inventoryService.checkAvailability(
            request.getItems()
        );
        if (!inventory.isAvailable()) {
            return ResponseEntity.status(400)
                .body(new OrderResponse("Items out of stock"));
        }
        
        // 4. Route to Order Service
        Order order = orderService.createOrder(
            user.getId(),
            request.getItems()
        );
        
        // 5. Return response
        return ResponseEntity.ok(new OrderResponse(order));
    }
}
```

**Step 3: Gateway Routes to Services**

```java
// Gateway calls Order Service
@Service
public class OrderServiceClient {
    private RestTemplate restTemplate;
    private String orderServiceUrl = "http://order-service:8080";
    
    public Order createOrder(String customerId, List<OrderItem> items) {
        CreateOrderRequest request = new CreateOrderRequest();
        request.setCustomerId(customerId);
        request.setItems(items);
        
        // Gateway routes to Order Service
        Order order = restTemplate.postForObject(
            orderServiceUrl + "/api/orders",
            request,
            Order.class
        );
        
        return order;
    }
}

// Gateway calls Inventory Service
@Service
public class InventoryServiceClient {
    private RestTemplate restTemplate;
    private String inventoryServiceUrl = "http://inventory-service:8080";
    
    public InventoryStatus checkAvailability(List<OrderItem> items) {
        CheckAvailabilityRequest request = new CheckAvailabilityRequest();
        request.setItems(items);
        
        // Gateway routes to Inventory Service
        InventoryStatus status = restTemplate.postForObject(
            inventoryServiceUrl + "/api/inventory/check",
            request,
            InventoryStatus.class
        );
        
        return status;
    }
}
```

**Step 4: Services Process Request**

```java
// Order Service receives request from gateway
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody CreateOrderRequest request) {
        Order order = orderService.create(request);
        return ResponseEntity.ok(order);
    }
}

// Inventory Service receives request from gateway
@RestController
@RequestMapping("/api/inventory")
public class InventoryController {
    
    @PostMapping("/check")
    public ResponseEntity<InventoryStatus> checkAvailability(
            @RequestBody CheckAvailabilityRequest request) {
        InventoryStatus status = inventoryService.check(request.getItems());
        return ResponseEntity.ok(status);
    }
}
```

**Step 5: Gateway Aggregates and Returns**

```java
// Gateway aggregates responses and returns to client
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(...) {
    // Check inventory
    InventoryStatus inventory = inventoryService.checkAvailability(items);
    
    // Create order
    Order order = orderService.createOrder(customerId, items);
    
    // Aggregate response
    OrderResponse response = new OrderResponse();
    response.setOrderId(order.getId());
    response.setStatus(order.getStatus());
    response.setItems(order.getItems());
    response.setInventoryStatus(inventory);
    
    return ResponseEntity.ok(response);
}
```

**Client receives:**

```json
{
  "orderId": "order-123",
  "status": "PENDING",
  "items": [
    {"productId": "456", "quantity": 2},
    {"productId": "789", "quantity": 1}
  ],
  "inventoryStatus": {
    "available": true,
    "reserved": true
  }
}
```

**Key Points:**
- Client only knows gateway URL
- Gateway handles authentication
- Gateway aggregates multiple services
- Gateway returns unified response
- Services stay hidden from client

---

## 5️⃣ How Engineers Actually Use This in Production

### Real-World Implementation: Spring Cloud Gateway

**Architecture:**

```
Clients (Mobile, Web, Admin)
  ↓
Spring Cloud Gateway
  ├── Authentication
  ├── Rate Limiting
  ├── Request Routing
  ├── Response Aggregation
  └── Protocol Translation
  ↓
Microservices (Order, Payment, Inventory, etc.)
```

**Gateway Configuration:**

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        # Route to Order Service
        - id: order-service
          uri: lb://order-service  # Load balanced
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1  # Remove /api prefix
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
        
        # Route to Payment Service
        - id: payment-service
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
          filters:
            - StripPrefix=1
        
        # Route to Inventory Service
        - id: inventory-service
          uri: lb://inventory-service
          predicates:
            - Path=/api/inventory/**
          filters:
            - StripPrefix=1
```

**Gateway Application:**

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("order-service", r -> r
                .path("/api/orders/**")
                .uri("lb://order-service"))
            .route("payment-service", r -> r
                .path("/api/payments/**")
                .uri("lb://payment-service"))
            .build();
    }
    
    @Bean
    public GlobalFilter customGlobalFilter() {
        return (exchange, chain) -> {
            // Add authentication
            String token = exchange.getRequest()
                .getHeaders()
                .getFirst("Authorization");
            
            if (token == null || !isValidToken(token)) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }
            
            return chain.filter(exchange);
        };
    }
}
```

**Request Aggregation (Custom Route):**

```java
@RestController
@RequestMapping("/api")
public class AggregationController {
    
    @Autowired
    private OrderServiceClient orderClient;
    
    @Autowired
    private PaymentServiceClient paymentClient;
    
    @Autowired
    private ShippingServiceClient shippingClient;
    
    @GetMapping("/order-details/{orderId}")
    public CompletableFuture<OrderDetailsResponse> getOrderDetails(
            @PathVariable String orderId) {
        
        // Call services in parallel
        CompletableFuture<Order> orderFuture = 
            CompletableFuture.supplyAsync(() -> orderClient.getOrder(orderId));
        
        CompletableFuture<Payment> paymentFuture = 
            CompletableFuture.supplyAsync(() -> paymentClient.getPaymentByOrder(orderId));
        
        CompletableFuture<Shipping> shippingFuture = 
            CompletableFuture.supplyAsync(() -> shippingClient.getShippingByOrder(orderId));
        
        // Aggregate responses
        return CompletableFuture.allOf(orderFuture, paymentFuture, shippingFuture)
            .thenApply(v -> {
                OrderDetailsResponse response = new OrderDetailsResponse();
                response.setOrder(orderFuture.join());
                response.setPayment(paymentFuture.join());
                response.setShipping(shippingFuture.join());
                return response;
            });
    }
}
```

### Netflix Zuul Implementation

Netflix uses Zuul API Gateway:

1. **Routing**: Routes requests to appropriate services
2. **Authentication**: Validates OAuth tokens
3. **Rate Limiting**: Limits requests per client
4. **Aggregation**: Aggregates responses from multiple services
5. **Monitoring**: Tracks all requests/responses

### Amazon API Gateway

Amazon uses API Gateway for:

1. **REST APIs**: Exposes REST endpoints
2. **WebSocket**: Handles WebSocket connections
3. **Lambda Integration**: Routes to serverless functions
4. **Request Transformation**: Transforms requests/responses
5. **Caching**: Caches responses for performance

### Kong API Gateway

Kong is a popular open-source API Gateway:

1. **Plugin System**: Extensible with plugins
2. **Rate Limiting**: Built-in rate limiting
3. **Authentication**: OAuth, JWT, API keys
4. **Load Balancing**: Load balances across services
5. **Monitoring**: Built-in monitoring and logging

---

## 6️⃣ How to Implement or Apply It

### Step-by-Step: Setting Up Spring Cloud Gateway

**Step 1: Add Dependencies**

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
    </dependency>
</dependencies>
```

**Step 2: Configure Gateway**

```yaml
# application.yml
server:
  port: 8080

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
                redis-rate-limiter.requestedTokens: 1
        
        - id: payment-service
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
          filters:
            - StripPrefix=1

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
```

**Step 3: Create Gateway Application**

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            String token = exchange.getRequest()
                .getHeaders()
                .getFirst("Authorization");
            return Mono.just(token != null ? token : "anonymous");
        };
    }
}
```

**Step 4: Add Authentication Filter**

```java
@Component
public class AuthenticationFilter implements GatewayFilter, Ordered {
    
    @Autowired
    private JwtTokenValidator tokenValidator;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest()
            .getHeaders()
            .getFirst("Authorization");
        
        if (token == null || !token.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        
        String jwt = token.substring(7);
        if (!tokenValidator.validate(jwt)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        
        // Add user info to headers for downstream services
        String userId = tokenValidator.getUserId(jwt);
        ServerHttpRequest modifiedRequest = exchange.getRequest().mutate()
            .header("X-User-Id", userId)
            .build();
        
        return chain.filter(exchange.mutate().request(modifiedRequest).build());
    }
    
    @Override
    public int getOrder() {
        return -100;
    }
}
```

**Step 5: Add Request Aggregation**

```java
@RestController
@RequestMapping("/api")
public class OrderAggregationController {
    
    @Autowired
    private WebClient.Builder webClientBuilder;
    
    @GetMapping("/order-details/{orderId}")
    public Mono<OrderDetailsResponse> getOrderDetails(@PathVariable String orderId) {
        WebClient webClient = webClientBuilder.build();
        
        Mono<Order> orderMono = webClient.get()
            .uri("http://order-service/api/orders/{orderId}", orderId)
            .retrieve()
            .bodyToMono(Order.class);
        
        Mono<Payment> paymentMono = webClient.get()
            .uri("http://payment-service/api/payments/order/{orderId}", orderId)
            .retrieve()
            .bodyToMono(Payment.class);
        
        Mono<Shipping> shippingMono = webClient.get()
            .uri("http://shipping-service/api/shipping/order/{orderId}", orderId)
            .retrieve()
            .bodyToMono(Shipping.class);
        
        return Mono.zip(orderMono, paymentMono, shippingMono)
            .map(tuple -> {
                OrderDetailsResponse response = new OrderDetailsResponse();
                response.setOrder(tuple.getT1());
                response.setPayment(tuple.getT2());
                response.setShipping(tuple.getT3());
                return response;
            });
    }
}
```

**Step 6: Test Gateway**

```bash
# Start gateway
./mvnw spring-boot:run

# Test routing
curl -X POST http://localhost:8080/api/orders \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"customerId": "123", "items": [...]}'

# Test aggregation
curl http://localhost:8080/api/order-details/123 \
  -H "Authorization: Bearer <token>"
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Tradeoffs

**Advantages:**
- **Single Entry Point**: Clients only know gateway
- **Centralized Concerns**: Auth, rate limiting, logging in one place
- **Service Hiding**: Services stay internal
- **Protocol Translation**: Gateway handles protocol differences
- **Response Aggregation**: Reduces client round trips

**Disadvantages:**
- **Single Point of Failure**: Gateway failure affects all clients
- **Performance Bottleneck**: All traffic goes through gateway
- **Complexity**: Additional component to maintain
- **Latency**: Extra hop adds latency
- **Scaling Challenge**: Gateway must scale with traffic

### Common Pitfalls

**Pitfall 1: Gateway as Single Point of Failure**

```java
// BAD: Single gateway instance
API Gateway (Single Instance)
  ↓
All Services
```

**Solution**: Run gateway in cluster, use load balancer.

**Pitfall 2: Too Much Logic in Gateway**

```java
// BAD: Business logic in gateway
@PostMapping("/orders")
public OrderResponse createOrder(...) {
    // Business logic should be in services
    if (order.getTotal() > 1000) {
        // Apply discount - this is business logic!
        order.setDiscount(0.1);
    }
    return order;
}
```

**Solution**: Gateway should route, not contain business logic.

**Pitfall 3: No Caching**

```java
// BAD: No caching, hits services every time
@GetMapping("/products/{id}")
public Product getProduct(@PathVariable String id) {
    return productService.getProduct(id);  // Always calls service
}
```

**Solution**: Cache responses in gateway.

**Pitfall 4: Synchronous Aggregation**

```java
// BAD: Sequential calls
Order order = orderService.getOrder(id);
Payment payment = paymentService.getPayment(orderId);
Shipping shipping = shippingService.getShipping(orderId);
// Slow: 3 sequential calls
```

**Solution**: Use parallel/async aggregation.

**Pitfall 5: No Rate Limiting**

```java
// BAD: No rate limiting
@PostMapping("/orders")
public Order createOrder(...) {
    // No rate limiting, can be abused
    return orderService.createOrder(...);
}
```

**Solution**: Implement rate limiting per client.

### Performance Considerations

**Gateway Overhead:**
- Each request adds gateway processing time
- For high-throughput, optimize gateway code
- Consider caching frequently accessed data

**Aggregation Overhead:**
- Aggregating multiple services adds latency
- Use async/parallel calls
- Consider if aggregation is necessary

**Scaling:**
- Gateway must scale with traffic
- Use horizontal scaling
- Consider regional gateways for global systems

---

## 8️⃣ When NOT to Use This

### Anti-Patterns and Misuse Cases

**Don't use API Gateway for:**

**1. Service-to-Service Communication**

Service-to-service calls should use service discovery, not gateway:

```java
// BAD: Service calls another service through gateway
Order Service → API Gateway → Payment Service
// Extra hop, unnecessary

// GOOD: Service calls another service directly
Order Service → Payment Service (via service discovery)
```

**2. Internal-Only Services**

If services are never called by external clients, gateway is overkill:

```java
// Internal batch processing service
// Never called by clients
// No need for gateway
```

**3. Simple Systems**

With 2-3 services and one client type, gateway might be overkill:

```java
// Simple system
Web App → Order Service
Web App → Payment Service
// Might not need gateway
```

### Situations Where This is Overkill

**Single Client, Few Services:**
- One client type, 2-3 services
- Gateway adds complexity without much benefit

**Serverless Architecture:**
- Serverless functions handle routing
- API Gateway might be redundant

### Better Alternatives for Specific Scenarios

**Service Mesh:**
- For advanced traffic management, security
- Service mesh handles routing, gateway might be redundant

**Direct Service Access:**
- For internal services
- Use service discovery instead

---

## 9️⃣ Comparison with Alternatives

### API Gateway vs Service Mesh

| Aspect | API Gateway | Service Mesh |
|--------|-------------|--------------|
| **Purpose** | Client-to-service communication | Service-to-service communication |
| **Location** | Edge of network | Between services |
| **Protocol** | HTTP/REST primarily | Any protocol |
| **Client** | External clients | Internal services |
| **Features** | Auth, rate limiting, aggregation | mTLS, traffic management, observability |

**Use both**: API Gateway for external clients, Service Mesh for service-to-service.

### API Gateway vs Load Balancer

**API Gateway:**
- Application-level routing
- Handles auth, rate limiting, aggregation
- More features, more complexity

**Load Balancer:**
- Network-level routing
- Simple load distribution
- Less features, less complexity

**Use both**: Load balancer in front of gateway cluster.

### API Gateway vs Reverse Proxy

**API Gateway:**
- Application-aware
- Handles business logic (aggregation, transformation)
- More features

**Reverse Proxy:**
- Network-level
- Simple forwarding
- Less features

**API Gateway is reverse proxy with more features.**

---

## 🔟 Interview Follow-up Questions WITH Answers

### L4 Level Questions

**Q1: What is API Gateway in microservices?**

**Answer**: API Gateway is a single entry point for all client requests to microservices. It sits between clients and services, handling routing, authentication, rate limiting, request aggregation, and protocol translation. Instead of clients calling services directly, clients call the gateway, and the gateway routes to appropriate services. This simplifies client code, centralizes cross-cutting concerns, and hides service complexity from clients.

**Q2: What are the main benefits of API Gateway?**

**Answer**:
1. **Single Entry Point**: Clients only know gateway URL, not all services
2. **Centralized Concerns**: Auth, rate limiting, logging in one place
3. **Service Hiding**: Services stay internal, not exposed to clients
4. **Protocol Translation**: Gateway handles different protocols (REST, gRPC)
5. **Response Aggregation**: Gateway can aggregate multiple service responses
6. **Simplified Clients**: Clients don't need to know about service discovery, multiple endpoints

**Q3: What's the difference between API Gateway and Load Balancer?**

**Answer**: Load Balancer distributes traffic across multiple instances of the same service (network-level). API Gateway routes requests to different services based on path/headers (application-level), and also handles auth, rate limiting, aggregation. API Gateway is more feature-rich but more complex. Often used together: Load Balancer in front of API Gateway cluster.

### L5 Level Questions

**Q4: How do you implement request aggregation in API Gateway?**

**Answer**: Request aggregation combines responses from multiple services into a single response:

1. **Parallel Calls**: Call multiple services in parallel (async)
2. **Wait for All**: Wait for all responses
3. **Aggregate**: Combine responses into single response
4. **Return**: Return aggregated response to client

Example:
```java
@GetMapping("/order-details/{orderId}")
public Mono<OrderDetails> getOrderDetails(@PathVariable String orderId) {
    Mono<Order> order = orderService.getOrder(orderId);
    Mono<Payment> payment = paymentService.getPayment(orderId);
    Mono<Shipping> shipping = shippingService.getShipping(orderId);
    
    return Mono.zip(order, payment, shipping)
        .map(tuple -> {
            OrderDetails details = new OrderDetails();
            details.setOrder(tuple.getT1());
            details.setPayment(tuple.getT2());
            details.setShipping(tuple.getT3());
            return details;
        });
}
```

**Q5: How do you handle authentication in API Gateway?**

**Answer**:
1. **Extract Token**: Extract JWT/OAuth token from request header
2. **Validate Token**: Validate token (check signature, expiration)
3. **Extract User Info**: Extract user ID, roles from token
4. **Add Headers**: Add user info to headers for downstream services
5. **Reject Invalid**: Reject requests with invalid/missing tokens

Example:
```java
@Component
public class AuthFilter implements GatewayFilter {
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = extractToken(exchange);
        if (!validateToken(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        String userId = extractUserId(token);
        ServerHttpRequest modified = exchange.getRequest().mutate()
            .header("X-User-Id", userId)
            .build();
        return chain.filter(exchange.mutate().request(modified).build());
    }
}
```

**Q6: How do you prevent API Gateway from becoming a bottleneck?**

**Answer**:
1. **Horizontal Scaling**: Run multiple gateway instances, load balance
2. **Caching**: Cache responses to reduce service calls
3. **Async Processing**: Use async/reactive patterns for aggregation
4. **Connection Pooling**: Reuse connections to services
5. **Rate Limiting**: Prevent abuse, protect services
6. **Regional Gateways**: Deploy gateways per region for global systems
7. **Optimize Code**: Profile and optimize gateway code
8. **Circuit Breakers**: Fail fast if services are down

### L6 Level Questions

**Q7: You're designing API Gateway for a system with 100 microservices and 5 client types. How do you structure it?**

**Answer**:

**Option 1: Single Gateway (Simple)**
- One gateway routes to all services
- Simple but can become bottleneck
- Good for small-medium scale

**Option 2: Backend for Frontend (BFF) Pattern (Recommended)**
- Separate gateway per client type:
  - Mobile Gateway
  - Web Gateway
  - Admin Gateway
  - Partner API Gateway
  - Public API Gateway
- Each optimized for its client
- Better performance, clearer separation

**Option 3: Service-Oriented Gateways**
- Gateway per service group:
  - Order Gateway (Order, Payment, Shipping)
  - Product Gateway (Product, Inventory, Catalog)
  - User Gateway (User, Auth, Profile)
- Better isolation, independent scaling

**Recommendation:**
- Use BFF pattern: 5 gateways (one per client type)
- Each gateway routes to relevant services
- Use service discovery for routing
- Implement caching, rate limiting per gateway
- Monitor each gateway independently

**Q8: How do you handle API versioning in API Gateway?**

**Answer**:
1. **URL Versioning**: `/v1/orders`, `/v2/orders`
2. **Header Versioning**: `Accept: application/vnd.api.v1+json`
3. **Route to Services**: Gateway routes based on version
4. **Service Mapping**: Map versions to service instances
5. **Deprecation**: Handle deprecated versions gracefully

Example:
```yaml
routes:
  - id: order-service-v1
    uri: lb://order-service-v1
    predicates:
      - Path=/v1/orders/**
  
  - id: order-service-v2
    uri: lb://order-service-v2
    predicates:
      - Path=/v2/orders/**
```

**Q9: Compare API Gateway with Service Mesh. When would you use each?**

**Answer**:

**API Gateway:**
- **Purpose**: Client-to-service communication
- **Location**: Edge of network
- **Clients**: External (mobile, web, partners)
- **Features**: Auth, rate limiting, aggregation, protocol translation
- **Use Case**: Single entry point for external clients

**Service Mesh:**
- **Purpose**: Service-to-service communication
- **Location**: Between services (sidecar)
- **Clients**: Internal services
- **Features**: mTLS, traffic management, observability, circuit breaking
- **Use Case**: Advanced service-to-service communication

**When to use:**
- **API Gateway**: For external clients, need aggregation, protocol translation
- **Service Mesh**: For service-to-service, need advanced traffic management, security
- **Both**: Large systems use both - Gateway for external, Mesh for internal

**Example:**
- Mobile app → API Gateway → Services (Gateway handles auth, aggregation)
- Order Service → Payment Service (Service Mesh handles mTLS, retries)

---

## 1️⃣1️⃣ One Clean Mental Summary

API Gateway is like a hotel reception desk that provides a single entry point for all client requests to microservices. Clients only know the gateway (reception), not individual services (departments). The gateway routes requests to appropriate services, handles authentication and rate limiting, aggregates responses from multiple services, and translates between protocols. This simplifies client code, centralizes cross-cutting concerns, and hides service complexity. The gateway sits at the edge of your network, between clients and services. Use it when you have multiple client types, need request aggregation, want to centralize auth/rate limiting, or need to hide services from clients. Don't use it for service-to-service communication (use service discovery instead) or for simple systems with few services. The main tradeoff is that it becomes a single point of failure and potential bottleneck, so you must scale it horizontally and optimize its performance. Common patterns include Backend for Frontend (BFF) where different client types get their own optimized gateway, and request aggregation where the gateway combines responses from multiple services into a single response.

