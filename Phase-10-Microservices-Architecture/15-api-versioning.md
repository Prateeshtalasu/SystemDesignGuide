# API Versioning in Microservices

## 0️⃣ Prerequisites

Before diving into this topic, you need to understand:

- **REST APIs**: HTTP methods, status codes, request/response patterns
- **Microservices Architecture**: Service independence and evolution (Phase 10, Topic 1)
- **API Design**: REST principles, resource design (Phase 2, Topic 12)
- **HTTP Headers**: Content-Type, Accept, custom headers
- **Backward Compatibility**: Understanding of breaking vs non-breaking changes

**Quick refresher**: In microservices, services evolve independently. When a service changes its API, existing clients might break. API versioning allows services to evolve while maintaining compatibility with existing clients. Different versioning strategies (URL, header, query parameter) have different tradeoffs in terms of discoverability, caching, and client complexity.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

In a microservices architecture, services evolve over time:

```
Order Service v1.0:
  GET /api/orders/{id}
  Returns: { orderId, customerId, items[], total }

Order Service v2.0 (New Requirements):
  GET /api/orders/{id}
  Returns: { orderId, customerId, items[], total, discount, tax }
  // Added new fields: discount, tax
```

**The Problem**: How do you handle API changes when:
- Existing clients depend on old API format
- New clients need new API features
- Services need to evolve without breaking clients
- Multiple client versions exist simultaneously
- Services deploy independently

### What Systems Looked Like Without Versioning

**Breaking Changes (Client Failures):**

```java
// Order Service changes API
// OLD: GET /api/orders/{id} returns { orderId, customerId, total }
// NEW: GET /api/orders/{id} returns { id, customer, amount }

// Mobile App (expects old format)
Order order = restTemplate.getForObject("/api/orders/123", Order.class);
String orderId = order.getOrderId();  // NULL! Field renamed to 'id'
// App crashes
```

**Problems:**
1. **Client Breaks**: Existing clients fail when API changes
2. **Forced Updates**: All clients must update simultaneously
3. **Deployment Coordination**: Services and clients must deploy together
4. **Risk**: Breaking changes cause production outages

### What Breaks Without Versioning

**At Scale:**

```
50 microservices
× 10 client applications each
= 500 integration points
× API changes every month
= Constant breakage and coordination overhead
```

**Real-World Failures:**

- **Field Rename**: Service renames `orderId` to `id`. All clients break. Production outage.

- **Field Removal**: Service removes `customerName` field. Clients expecting it get null. UI shows blank names.

- **Response Structure Change**: Service changes from flat to nested structure. Clients can't parse. Errors everywhere.

- **Required Field Added**: Service adds required field `discount`. Old clients don't send it. Validation fails. Orders rejected.

- **Deployment Coordination**: Service deploys new API. Clients not ready. Service must rollback. Deployment blocked.

### Real Examples of the Problem

**Twitter API (Early Days)**:
- Changed API without versioning
- Broke all third-party clients
- Caused major outages
- Adopted versioning: `/1.1/statuses/...`

**Stripe API**:
- Handles millions of API calls
- Must support multiple API versions
- Uses date-based versioning: `API-Version: 2020-08-27`
- Maintains backward compatibility

**GitHub API**:
- Multiple versions: v3, v4 (GraphQL)
- URL versioning: `/v3/repos/...`
- Maintains old versions for years

---

## 2️⃣ Intuition and Mental Model

### The Restaurant Menu Analogy

Think of **API Versioning** like restaurant menu versions.

**Without Versioning (Breaking Changes):**

```
Restaurant Menu v1.0:
  - Pizza: $10
  - Burger: $8

Restaurant Menu v2.0 (Changed):
  - Pizza: $12 (price changed)
  - Burger: REMOVED
  - Pasta: $11 (new item)

Customer orders "Burger" (expects v1.0):
  - Restaurant: "We don't have that anymore"
  - Customer: Confused, order fails
```

**With Versioning (Multiple Menus):**

```
Restaurant Menu v1.0 (Still Available):
  - Pizza: $10
  - Burger: $8

Restaurant Menu v2.0 (New Menu):
  - Pizza: $12
  - Pasta: $11

Customer orders:
  - "I want v1.0 menu" → Gets v1.0
  - "I want v2.0 menu" → Gets v2.0
  - Both work simultaneously
```

**API Versioning Components:**

- **Menu Versions**: API versions (v1, v2, v3)
- **Customer Choice**: Client specifies version
- **Simultaneous Support**: Restaurant serves multiple menus
- **Gradual Migration**: Customers migrate to new menu over time

### The Key Mental Model

**API Versioning is like maintaining multiple menu versions:**

- **Services** maintain multiple API versions (like multiple menus)
- **Clients** specify which version they want (like choosing menu)
- **Evolution** happens without breaking clients (new menu doesn't break old customers)
- **Migration** happens gradually (customers migrate to new menu over time)

This analogy helps us understand:
- Why multiple versions exist (different customers need different menus)
- Why clients specify version (customer chooses menu)
- Why services maintain old versions (old customers still come)
- Why migration is gradual (can't force all customers to new menu)

---

## 3️⃣ How It Works Internally

API Versioning follows these strategies:

### Strategy 1: URL Versioning

```
┌─────────────────────────────────────────────────────────┐
│              URL VERSIONING                               │
└─────────────────────────────────────────────────────────┘

Client
  ↓
GET /v1/orders/123  (v1 API)
  ↓
Order Service
  ├── Routes to v1 handler
  └── Returns v1 response format

Client
  ↓
GET /v2/orders/123  (v2 API)
  ↓
Order Service
  ├── Routes to v2 handler
  └── Returns v2 response format
```

**Flow:**
1. Client includes version in URL: `/v1/orders/123`
2. Service routes based on version path
3. Service returns version-specific response
4. Different versions can have different implementations

### Strategy 2: Header Versioning

```
┌─────────────────────────────────────────────────────────┐
│              HEADER VERSIONING                           │
└─────────────────────────────────────────────────────────┘

Client
  ↓
GET /api/orders/123
Headers: API-Version: v1
  ↓
Order Service
  ├── Reads version from header
  ├── Routes to v1 handler
  └── Returns v1 response format

Client
  ↓
GET /api/orders/123
Headers: API-Version: v2
  ↓
Order Service
  ├── Reads version from header
  ├── Routes to v2 handler
  └── Returns v2 response format
```

**Flow:**
1. Client includes version in header: `API-Version: v1`
2. Service reads header, routes to version handler
3. Service returns version-specific response
4. URL stays same, version in header

### Strategy 3: Content Negotiation

```
┌─────────────────────────────────────────────────────────┐
│              CONTENT NEGOTIATION                         │
└─────────────────────────────────────────────────────────┘

Client
  ↓
GET /api/orders/123
Accept: application/vnd.api.v1+json
  ↓
Order Service
  ├── Reads Accept header
  ├── Routes to v1 handler
  └── Returns v1 response format

Client
  ↓
GET /api/orders/123
Accept: application/vnd.api.v2+json
  ↓
Order Service
  ├── Reads Accept header
  ├── Routes to v2 handler
  └── Returns v2 response format
```

**Flow:**
1. Client specifies version in Accept header
2. Service reads Accept header, routes to version
3. Service returns version-specific content type
4. Uses HTTP content negotiation standard

---

## 4️⃣ Simulation-First Explanation

Let's trace API versioning through a request:

### Scenario: Client Calls Order Service with Different Versions

**Setup:**
- Order Service: Supports v1 and v2
- Mobile App: Uses v1
- Web App: Uses v2

**Step 1: Mobile App Calls v1 API (URL Versioning)**

```http
GET /v1/orders/123
Authorization: Bearer <token>
```

```java
// Order Service - URL Versioning
@RestController
public class OrderController {
    
    // v1 API
    @GetMapping("/v1/orders/{orderId}")
    public ResponseEntity<OrderV1> getOrderV1(@PathVariable String orderId) {
        Order order = orderService.getOrder(orderId);
        
        // Return v1 format
        OrderV1 response = new OrderV1();
        response.setOrderId(order.getId());
        response.setCustomerId(order.getCustomerId());
        response.setTotal(order.getTotal());
        // v1 doesn't have discount, tax
        
        return ResponseEntity.ok(response);
    }
    
    // v2 API
    @GetMapping("/v2/orders/{orderId}")
    public ResponseEntity<OrderV2> getOrderV2(@PathVariable String orderId) {
        Order order = orderService.getOrder(orderId);
        
        // Return v2 format
        OrderV2 response = new OrderV2();
        response.setId(order.getId());  // Field renamed
        response.setCustomer(order.getCustomer());  // Nested object
        response.setTotal(order.getTotal());
        response.setDiscount(order.getDiscount());  // New field
        response.setTax(order.getTax());  // New field
        
        return ResponseEntity.ok(response);
    }
}
```

**Mobile App receives v1 response:**

```json
{
  "orderId": "123",
  "customerId": "456",
  "total": 100.00
}
```

**Step 2: Web App Calls v2 API**

```http
GET /v2/orders/123
Authorization: Bearer <token>
```

**Web App receives v2 response:**

```json
{
  "id": "123",
  "customer": {
    "id": "456",
    "name": "John Doe"
  },
  "total": 100.00,
  "discount": 10.00,
  "tax": 9.00
}
```

**Step 3: Header Versioning Example**

```http
GET /api/orders/123
Authorization: Bearer <token>
API-Version: v1
```

```java
// Order Service - Header Versioning
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @GetMapping("/{orderId}")
    public ResponseEntity<?> getOrder(
            @PathVariable String orderId,
            @RequestHeader(value = "API-Version", defaultValue = "v1") String version) {
        
        Order order = orderService.getOrder(orderId);
        
        if ("v1".equals(version)) {
            return ResponseEntity.ok(toOrderV1(order));
        } else if ("v2".equals(version)) {
            return ResponseEntity.ok(toOrderV2(order));
        } else {
            return ResponseEntity.badRequest().build();
        }
    }
    
    private OrderV1 toOrderV1(Order order) {
        OrderV1 v1 = new OrderV1();
        v1.setOrderId(order.getId());
        v1.setCustomerId(order.getCustomerId());
        v1.setTotal(order.getTotal());
        return v1;
    }
    
    private OrderV2 toOrderV2(Order order) {
        OrderV2 v2 = new OrderV2();
        v2.setId(order.getId());
        v2.setCustomer(toCustomer(order.getCustomer()));
        v2.setTotal(order.getTotal());
        v2.setDiscount(order.getDiscount());
        v2.setTax(order.getTax());
        return v2;
    }
}
```

**Step 4: Content Negotiation Example**

```http
GET /api/orders/123
Authorization: Bearer <token>
Accept: application/vnd.api.v1+json
```

```java
// Order Service - Content Negotiation
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @GetMapping(value = "/{orderId}", produces = {
        "application/vnd.api.v1+json",
        "application/vnd.api.v2+json"
    })
    public ResponseEntity<?> getOrder(
            @PathVariable String orderId,
            @RequestHeader("Accept") String accept) {
        
        Order order = orderService.getOrder(orderId);
        
        if (accept.contains("v1")) {
            return ResponseEntity.ok()
                .contentType(MediaType.parseMediaType("application/vnd.api.v1+json"))
                .body(toOrderV1(order));
        } else if (accept.contains("v2")) {
            return ResponseEntity.ok()
                .contentType(MediaType.parseMediaType("application/vnd.api.v2+json"))
                .body(toOrderV2(order));
        } else {
            return ResponseEntity.notAcceptable().build();
        }
    }
}
```

**Key Points:**
- Multiple versions coexist
- Clients specify version
- Service routes to version handler
- Different versions can have different formats
- Old clients continue working

---

## 5️⃣ How Engineers Actually Use This in Production

### Real-World Implementation: Spring Boot Versioning

**Architecture:**

```
Order Service
  ├── v1 Controllers
  ├── v2 Controllers
  └── Shared Service Layer
```

**URL Versioning Implementation:**

```java
// v1 Controller
@RestController
@RequestMapping("/v1/orders")
public class OrderV1Controller {
    
    @Autowired
    private OrderService orderService;
    
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderV1Response> getOrder(@PathVariable String orderId) {
        Order order = orderService.getOrder(orderId);
        return ResponseEntity.ok(toV1Response(order));
    }
    
    private OrderV1Response toV1Response(Order order) {
        OrderV1Response response = new OrderV1Response();
        response.setOrderId(order.getId());
        response.setCustomerId(order.getCustomerId());
        response.setItems(order.getItems());
        response.setTotal(order.getTotal());
        return response;
    }
}

// v2 Controller
@RestController
@RequestMapping("/v2/orders")
public class OrderV2Controller {
    
    @Autowired
    private OrderService orderService;
    
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderV2Response> getOrder(@PathVariable String orderId) {
        Order order = orderService.getOrder(orderId);
        return ResponseEntity.ok(toV2Response(order));
    }
    
    private OrderV2Response toV2Response(Order order) {
        OrderV2Response response = new OrderV2Response();
        response.setId(order.getId());
        response.setCustomer(toCustomerDto(order.getCustomer()));
        response.setItems(order.getItems().stream()
            .map(this::toItemDto)
            .collect(Collectors.toList()));
        response.setTotal(order.getTotal());
        response.setDiscount(order.getDiscount());
        response.setTax(order.getTax());
        return response;
    }
}
```

**Header Versioning Implementation:**

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @GetMapping("/{orderId}")
    public ResponseEntity<?> getOrder(
            @PathVariable String orderId,
            @RequestHeader(value = "API-Version", defaultValue = "v1") String version) {
        
        Order order = orderService.getOrder(orderId);
        
        return switch (version) {
            case "v1" -> ResponseEntity.ok(toV1Response(order));
            case "v2" -> ResponseEntity.ok(toV2Response(order));
            default -> ResponseEntity.badRequest()
                .body(new ErrorResponse("Unsupported API version: " + version));
        };
    }
}
```

**Version Interceptor (Centralized):**

```java
@Component
public class ApiVersionInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                           HttpServletResponse response, 
                           Object handler) {
        String version = extractVersion(request);
        request.setAttribute("apiVersion", version);
        return true;
    }
    
    private String extractVersion(HttpServletRequest request) {
        // Try header first
        String headerVersion = request.getHeader("API-Version");
        if (headerVersion != null) {
            return headerVersion;
        }
        
        // Try URL path
        String path = request.getRequestURI();
        Matcher matcher = Pattern.compile("/v(\\d+)/").matcher(path);
        if (matcher.find()) {
            return "v" + matcher.group(1);
        }
        
        // Default to v1
        return "v1";
    }
}
```

**Configuration:**

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Autowired
    private ApiVersionInterceptor apiVersionInterceptor;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(apiVersionInterceptor);
    }
}
```

### Stripe Implementation

Stripe uses date-based versioning:

```http
GET /v1/customers
Stripe-Version: 2020-08-27
```

- Versions are dates, not numbers
- Clients specify version in header
- Old versions maintained for years
- New versions add features, maintain compatibility

### GitHub Implementation

GitHub uses URL versioning:

```http
GET /v3/repos/owner/repo
GET /v4 (GraphQL endpoint)
```

- v3: REST API
- v4: GraphQL API
- URL-based, easy to discover
- Both maintained simultaneously

### Twitter Implementation

Twitter uses URL versioning:

```http
GET /1.1/statuses/user_timeline.json
GET /2/tweets
```

- Numbered versions in URL
- Old versions maintained
- Clear version in path

---

## 6️⃣ How to Implement or Apply It

### Step-by-Step: Implementing URL Versioning

**Step 1: Create Version-Specific DTOs**

```java
// v1 DTOs
public class OrderV1Response {
    private String orderId;
    private String customerId;
    private List<OrderItem> items;
    private BigDecimal total;
    // Getters and setters
}

// v2 DTOs
public class OrderV2Response {
    private String id;  // Renamed from orderId
    private CustomerDto customer;  // Nested object
    private List<OrderItemDto> items;
    private BigDecimal total;
    private BigDecimal discount;  // New field
    private BigDecimal tax;  // New field
    // Getters and setters
}
```

**Step 2: Create Version Controllers**

```java
// v1 Controller
@RestController
@RequestMapping("/v1/orders")
@ApiVersion("v1")
public class OrderV1Controller {
    
    @Autowired
    private OrderService orderService;
    
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderV1Response> getOrder(@PathVariable String orderId) {
        Order order = orderService.getOrder(orderId);
        return ResponseEntity.ok(mapToV1(order));
    }
    
    private OrderV1Response mapToV1(Order order) {
        OrderV1Response response = new OrderV1Response();
        response.setOrderId(order.getId());
        response.setCustomerId(order.getCustomerId());
        response.setItems(order.getItems());
        response.setTotal(order.getTotal());
        return response;
    }
}

// v2 Controller
@RestController
@RequestMapping("/v2/orders")
@ApiVersion("v2")
public class OrderV2Controller {
    
    @Autowired
    private OrderService orderService;
    
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderV2Response> getOrder(@PathVariable String orderId) {
        Order order = orderService.getOrder(orderId);
        return ResponseEntity.ok(mapToV2(order));
    }
    
    private OrderV2Response mapToV2(Order order) {
        OrderV2Response response = new OrderV2Response();
        response.setId(order.getId());
        response.setCustomer(mapCustomer(order.getCustomer()));
        response.setItems(order.getItems().stream()
            .map(this::mapItem)
            .collect(Collectors.toList()));
        response.setTotal(order.getTotal());
        response.setDiscount(order.getDiscount());
        response.setTax(order.getTax());
        return response;
    }
}
```

**Step 3: Shared Service Layer**

```java
// Service layer is version-agnostic
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    public Order getOrder(String orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
    
    // Business logic doesn't change with version
}
```

**Step 4: Version Routing Configuration**

```java
@Configuration
public class ApiVersionConfig {
    
    @Bean
    public RouterFunction<ServerResponse> versionRouter() {
        return RouterFunctions.route()
            .path("/v1", this::v1Routes)
            .path("/v2", this::v2Routes)
            .build();
    }
    
    private RouterFunction<ServerResponse> v1Routes() {
        return RouterFunctions.route()
            .GET("/orders/{id}", this::getOrderV1)
            .build();
    }
    
    private RouterFunction<ServerResponse> v2Routes() {
        return RouterFunctions.route()
            .GET("/orders/{id}", this::getOrderV2)
            .build();
    }
}
```

**Step 5: Deprecation Handling**

```java
@RestController
@RequestMapping("/v1/orders")
@Deprecated
public class OrderV1Controller {
    
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderV1Response> getOrder(
            @PathVariable String orderId,
            HttpServletResponse response) {
        
        // Add deprecation header
        response.setHeader("Deprecation", "true");
        response.setHeader("Sunset", "Sat, 31 Dec 2024 23:59:59 GMT");
        response.setHeader("Link", "</v2/orders>; rel=\"successor-version\"");
        
        Order order = orderService.getOrder(orderId);
        return ResponseEntity.ok(mapToV1(order));
    }
}
```

**Step 6: Version Documentation**

```java
@RestController
@RequestMapping("/api")
public class ApiInfoController {
    
    @GetMapping("/versions")
    public ResponseEntity<ApiVersions> getVersions() {
        ApiVersions versions = new ApiVersions();
        versions.setCurrentVersion("v2");
        versions.setSupportedVersions(Arrays.asList("v1", "v2"));
        versions.setDeprecatedVersions(Arrays.asList("v1"));
        versions.setEndOfLife("v1", LocalDate.of(2024, 12, 31));
        return ResponseEntity.ok(versions);
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Tradeoffs: URL vs Header vs Content Negotiation

**URL Versioning:**

**Advantages:**
- Easy to discover (version in URL)
- Cacheable (different URLs, different cache keys)
- Clear in logs
- Easy to test

**Disadvantages:**
- URL changes with version
- Can't use same URL for different versions
- More routing complexity

**Header Versioning:**

**Advantages:**
- URL stays same
- Clean URLs
- Flexible

**Disadvantages:**
- Harder to discover
- Caching issues (same URL, different versions)
- Less visible in logs

**Content Negotiation:**

**Advantages:**
- HTTP standard
- Flexible
- Supports multiple content types

**Disadvantages:**
- Complex
- Less common
- Harder to understand

### Common Pitfalls

**Pitfall 1: Breaking Changes in Same Version**

```java
// BAD: Breaking change without new version
@GetMapping("/v1/orders/{orderId}")
public OrderV1Response getOrder(@PathVariable String orderId) {
    // Changed response structure without versioning
    return new OrderV1Response(order.getId(), ...);  // Breaking change!
}
```

**Solution**: Create new version for breaking changes.

**Pitfall 2: Too Many Versions**

```java
// BAD: Maintaining 10 versions
/v1/orders
/v2/orders
/v3/orders
...
/v10/orders
// Too many to maintain
```

**Solution**: Deprecate old versions aggressively, limit active versions.

**Pitfall 3: No Deprecation Strategy**

```java
// BAD: No deprecation notice
@GetMapping("/v1/orders/{orderId}")
public OrderV1Response getOrder(...) {
    // No deprecation headers
    // Clients don't know it's deprecated
}
```

**Solution**: Add deprecation headers, communicate timeline.

**Pitfall 4: Version in Business Logic**

```java
// BAD: Version logic in service
@Service
public class OrderService {
    public Order getOrder(String orderId, String version) {
        if ("v1".equals(version)) {
            // v1 logic
        } else {
            // v2 logic
        }
        // Business logic shouldn't know about versions
    }
}
```

**Solution**: Keep versioning in controllers, service layer version-agnostic.

**Pitfall 5: No Default Version**

```java
// BAD: No default, fails if version not specified
@GetMapping("/orders/{orderId}")
public OrderResponse getOrder(@PathVariable String orderId) {
    // What if client doesn't specify version?
    // Should fail or default?
}
```

**Solution**: Always have default version (usually latest).

### Performance Considerations

**Version Routing Overhead:**
- Version detection adds small overhead
- Minimize by caching version handlers
- Use efficient routing

**Multiple Version Maintenance:**
- Each version adds maintenance burden
- Limit number of active versions
- Deprecate aggressively

**Caching:**
- URL versioning: Different URLs, different caches (good)
- Header versioning: Same URL, need version-aware cache (complex)

---

## 8️⃣ When NOT to Use This

### Anti-Patterns and Misuse Cases

**Don't use versioning for:**

**1. Non-Breaking Changes**

If change is backward compatible, no versioning needed:

```java
// Adding optional field - backward compatible
// OLD: { orderId, total }
// NEW: { orderId, total, discount }  // discount is optional
// No versioning needed
```

**2. Internal Services**

For service-to-service communication, consider if versioning is worth complexity:

```java
// Internal service-to-service
// Might use contract testing instead
// Versioning adds complexity
```

**3. Frequent Changes**

If API changes weekly, versioning becomes maintenance burden:

```java
// API changes every week
// Creating new version each week is unsustainable
// Consider if API design is stable enough
```

### Situations Where This is Overkill

**Simple APIs:**
- Single client, simple API
- Versioning might be overkill
- Consider backward compatibility instead

**Prototype/MVP:**
- Early stage, API not stable
- Versioning adds complexity
- Wait until API stabilizes

### Better Alternatives for Specific Scenarios

**Backward Compatibility:**
- Instead of versioning, make changes backward compatible
- Add fields, don't remove
- Make new fields optional

**Feature Flags:**
- For A/B testing features
- Use feature flags instead of versions
- Simpler than versioning

**Contract Testing:**
- For service-to-service
- Use contract testing (Pact)
- Ensures compatibility without versioning

---

## 9️⃣ Comparison with Alternatives

### API Versioning vs Backward Compatibility

| Aspect | API Versioning | Backward Compatibility |
|--------|---------------|----------------------|
| **Breaking Changes** | Supported (new version) | Not supported |
| **Complexity** | Higher (multiple versions) | Lower (single version) |
| **Client Updates** | Gradual (migrate over time) | Immediate (all update) |
| **Maintenance** | Higher (maintain multiple) | Lower (maintain one) |
| **Use Case** | Breaking changes needed | Can avoid breaking changes |

**Use both**: Maintain backward compatibility when possible, version when breaking changes are necessary.

### API Versioning vs Feature Flags

**API Versioning:**
- Different API contracts
- Long-term support
- Client migration over time

**Feature Flags:**
- Same API, different behavior
- Short-term A/B testing
- Quick enable/disable

**Different purposes**: Versioning for API evolution, feature flags for behavior control.

### API Versioning vs Contract Testing

**API Versioning:**
- Multiple API contracts
- Client specifies version
- Long-term support

**Contract Testing:**
- Ensures compatibility
- No versioning needed
- Service-to-service focus

**Use together**: Version APIs, test contracts for each version.

---

## 🔟 Interview Follow-up Questions WITH Answers

### L4 Level Questions

**Q1: What is API versioning and why is it needed?**

**Answer**: API versioning is the practice of maintaining multiple versions of an API simultaneously. It's needed because services evolve over time, and breaking changes would break existing clients. By versioning, you can introduce new features and breaking changes in new versions while maintaining old versions for existing clients. This allows gradual migration rather than forcing all clients to update simultaneously.

**Q2: What are the main strategies for API versioning?**

**Answer**:
1. **URL Versioning**: Version in URL path (`/v1/orders`, `/v2/orders`)
   - Pros: Easy to discover, cacheable, clear in logs
   - Cons: URL changes with version

2. **Header Versioning**: Version in HTTP header (`API-Version: v1`)
   - Pros: Clean URLs, flexible
   - Cons: Harder to discover, caching complexity

3. **Content Negotiation**: Version in Accept header (`Accept: application/vnd.api.v1+json`)
   - Pros: HTTP standard, flexible
   - Cons: Complex, less common

**Q3: What's the difference between breaking and non-breaking changes?**

**Answer**:
- **Breaking Changes**: Changes that break existing clients
  - Removing fields
  - Renaming fields
  - Changing field types
  - Making optional fields required
  - Requires new version

- **Non-Breaking Changes**: Changes that don't break clients
  - Adding optional fields
  - Adding new endpoints
  - Adding new optional query parameters
  - Doesn't require versioning

### L5 Level Questions

**Q4: How do you implement URL-based versioning in Spring Boot?**

**Answer**:

```java
// v1 Controller
@RestController
@RequestMapping("/v1/orders")
public class OrderV1Controller {
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderV1Response> getOrder(@PathVariable String orderId) {
        Order order = orderService.getOrder(orderId);
        return ResponseEntity.ok(mapToV1(order));
    }
}

// v2 Controller
@RestController
@RequestMapping("/v2/orders")
public class OrderV2Controller {
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderV2Response> getOrder(@PathVariable String orderId) {
        Order order = orderService.getOrder(orderId);
        return ResponseEntity.ok(mapToV2(order));
    }
}

// Shared service layer (version-agnostic)
@Service
public class OrderService {
    public Order getOrder(String orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
}
```

**Q5: How do you handle deprecation of old API versions?**

**Answer**:
1. **Deprecation Headers**: Add HTTP headers indicating deprecation
   ```java
   response.setHeader("Deprecation", "true");
   response.setHeader("Sunset", "Sat, 31 Dec 2024 23:59:59 GMT");
   response.setHeader("Link", "</v2/orders>; rel=\"successor-version\"");
   ```

2. **Communication**: Notify clients about deprecation timeline
3. **Monitoring**: Track usage of deprecated versions
4. **Gradual Migration**: Help clients migrate to new version
5. **End of Life**: Set clear end-of-life date, stick to it

**Q6: How do you test multiple API versions?**

**Answer**:
1. **Version-Specific Tests**: Test each version independently
2. **Contract Tests**: Test that each version matches its contract
3. **Integration Tests**: Test version routing works correctly
4. **Backward Compatibility Tests**: Ensure old versions still work

Example:
```java
@Test
void testV1Api() {
    OrderV1Response response = restTemplate.getForObject(
        "/v1/orders/123", OrderV1Response.class);
    assertNotNull(response.getOrderId());
}

@Test
void testV2Api() {
    OrderV2Response response = restTemplate.getForObject(
        "/v2/orders/123", OrderV2Response.class);
    assertNotNull(response.getId());
}
```

### L6 Level Questions

**Q7: You're designing API versioning for a system with 50 microservices. How do you ensure consistency?**

**Answer**:

**Option 1: Centralized Versioning Strategy**
- Define versioning strategy in API Gateway
- Gateway routes based on version
- Services don't handle versioning
- Consistent across all services

**Option 2: Service-Level Versioning**
- Each service handles its own versioning
- Standardize versioning approach (URL vs header)
- Document versioning strategy
- Use shared libraries for version handling

**Option 3: Hybrid Approach (Recommended)**
- API Gateway handles external client versioning
- Services handle internal versioning if needed
- Standardize on URL versioning for consistency
- Centralized documentation of all versions

**Recommendation:**
- Use API Gateway for external clients (consistent entry point)
- Standardize on URL versioning (`/v1/`, `/v2/`)
- Document versioning strategy in API design guidelines
- Use versioning only when breaking changes are necessary
- Deprecate old versions aggressively (maintain max 2-3 versions)

**Q8: How do you handle versioning when a service needs to call another service with different versions?**

**Answer**:

**Option 1: Service Specifies Version**
- Calling service specifies version in request
- Called service routes to version handler
- Services can use different versions

**Option 2: Latest Version Default**
- Services always use latest version
- Simpler, but requires coordination
- Breaking changes affect all services

**Option 3: Version Negotiation**
- Services negotiate compatible version
- Complex, rarely used

**Recommendation:**
- Services use latest stable version by default
- For breaking changes, update calling services gradually
- Use contract testing to ensure compatibility
- Consider service mesh for version-aware routing

**Q9: Compare different versioning strategies. When would you choose each?**

**Answer**:

**URL Versioning:**
- **Best for**: Public APIs, REST APIs, when discoverability matters
- **Example**: GitHub (`/v3/repos`), Twitter (`/1.1/statuses`)
- **Pros**: Easy to discover, cacheable, clear in logs
- **Cons**: URL changes, routing complexity

**Header Versioning:**
- **Best for**: When you want clean URLs, internal APIs
- **Example**: Stripe (`Stripe-Version: 2020-08-27`)
- **Pros**: Clean URLs, flexible
- **Cons**: Harder to discover, caching complexity

**Content Negotiation:**
- **Best for**: When supporting multiple content types
- **Example**: APIs with JSON and XML
- **Pros**: HTTP standard, flexible
- **Cons**: Complex, less common

**Recommendation:**
- **Public APIs**: URL versioning (discoverability)
- **Internal APIs**: Header versioning (clean URLs) or URL versioning (consistency)
- **Multi-format APIs**: Content negotiation
- **Choose based on**: Discoverability needs, caching requirements, team preferences

---

## 1️⃣1️⃣ One Clean Mental Summary

API versioning is like maintaining multiple restaurant menu versions simultaneously. Services evolve, and breaking changes would break existing clients. Versioning allows you to introduce new features in new versions while maintaining old versions for existing clients. Clients specify which version they want (like choosing a menu), and services route to the appropriate version handler. The main strategies are URL versioning (version in path like `/v1/orders`), header versioning (version in header like `API-Version: v1`), and content negotiation (version in Accept header). URL versioning is most common because it's easy to discover and cacheable. The key is to version only when breaking changes are necessary, maintain backward compatibility when possible, and deprecate old versions aggressively to limit maintenance burden. Services should keep versioning in controllers (routing layer) and keep business logic version-agnostic. Use versioning for public APIs and when services evolve independently, but consider if backward compatibility or other approaches might be simpler for your use case.

