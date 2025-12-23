# Idempotency

## 0️⃣ Prerequisites

Before diving into idempotency, you should understand:

- **HTTP Methods** (GET, POST, PUT, DELETE): The basic operations in web APIs
- **Distributed Systems** (covered in Topic 01): Systems where multiple computers communicate over unreliable networks
- **Failure Modes** (covered in Topic 07): Network failures, timeouts, and retries

**Quick refresher on retries**: In distributed systems, when a request fails or times out, we often retry it. But here's the problem: did the first request actually succeed before the timeout? If it did, and we retry, we might execute the operation twice. Idempotency solves this problem.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

Imagine you're buying a concert ticket online. You click "Purchase," and the page hangs. After 30 seconds, you see a timeout error. Did your purchase go through?

You click "Purchase" again. Now you've been charged twice for the same ticket.

This is the **duplicate request problem**, and it happens constantly in distributed systems.

### What Systems Looked Like Before Idempotency

In early web applications:

1. **Double charges**: Users clicking "Submit" multiple times would be charged multiple times
2. **Duplicate orders**: Network retries would create multiple orders for the same purchase
3. **Data corruption**: Two identical update requests would apply the update twice
4. **Lost trust**: Users learned to be afraid of clicking buttons

### What Breaks Without It

**Without idempotency:**

- **Payment systems**: Double charges, triple charges, refund nightmares
- **Inventory systems**: Stock counts become incorrect
- **Messaging systems**: Same message delivered multiple times
- **Order systems**: Duplicate orders, angry customers
- **API integrations**: Partners retry failed requests, causing duplicates

### Real Examples of the Problem

**Stripe's Idempotency Key**:
Stripe, the payment processor, handles millions of transactions daily. They discovered early that network issues caused significant duplicate charges. Their solution? Require an "idempotency key" with every payment request. If they receive the same key twice, they return the original response instead of processing again.

**Amazon's Order Processing**:
Amazon processes hundreds of orders per second. During high-traffic events (Prime Day), network congestion causes timeouts. Without idempotency, retries would create duplicate orders. Amazon uses order tokens to ensure each order is processed exactly once.

**Uber's Trip Requests**:
When you request an Uber, your phone might retry if the network is slow. Without idempotency, you could end up with multiple drivers coming to pick you up.

---

## 2️⃣ Intuition and Mental Model

### The Light Switch Analogy

Think about a light switch:

**Non-idempotent operation: "Toggle the light"**
- First toggle: Light turns ON
- Second toggle: Light turns OFF
- Third toggle: Light turns ON
- Each operation changes the state

**Idempotent operation: "Turn the light ON"**
- First "turn ON": Light turns ON
- Second "turn ON": Light is already ON, stays ON
- Third "turn ON": Light is already ON, stays ON
- Multiple operations have the same effect as one

### The Formal Definition

An operation is **idempotent** if performing it multiple times has the same effect as performing it once.

Mathematically: `f(f(x)) = f(x)`

### HTTP Methods and Idempotency

The HTTP specification defines which methods should be idempotent:

```
┌──────────┬─────────────┬────────────────────────────────────┐
│ Method   │ Idempotent? │ Why                                │
├──────────┼─────────────┼────────────────────────────────────┤
│ GET      │ Yes         │ Reading doesn't change state       │
│ HEAD     │ Yes         │ Same as GET, no body               │
│ OPTIONS  │ Yes         │ Describes available methods        │
│ PUT      │ Yes         │ "Set to this value" (absolute)     │
│ DELETE   │ Yes         │ "Remove this" (already gone = OK)  │
│ POST     │ NO          │ "Create new" (each call = new)     │
│ PATCH    │ NO*         │ "Modify" (depends on operation)    │
└──────────┴─────────────┴────────────────────────────────────┘

* PATCH can be idempotent if designed carefully
```

---

## 3️⃣ How It Works Internally

### The Core Problem: At-Least-Once vs Exactly-Once

In distributed systems, you have three delivery guarantees:

1. **At-most-once**: Send once, don't retry. Might lose messages.
2. **At-least-once**: Retry until acknowledged. Might duplicate.
3. **Exactly-once**: Each message processed exactly once. The holy grail.

**The truth**: True exactly-once delivery is impossible in distributed systems (proven by the Two Generals Problem). But we can achieve **exactly-once semantics** by combining at-least-once delivery with idempotent operations.

```
At-least-once delivery + Idempotent operations = Exactly-once semantics
```

### How Idempotency Keys Work

The most common pattern for making operations idempotent:

```
┌─────────────────────────────────────────────────────────────┐
│                    IDEMPOTENCY KEY FLOW                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Client                          Server                      │
│    │                               │                         │
│    │  POST /payment                │                         │
│    │  Idempotency-Key: abc123      │                         │
│    │  {amount: $100}               │                         │
│    │ ─────────────────────────────>│                         │
│    │                               │                         │
│    │                     ┌─────────┴─────────┐               │
│    │                     │ Check: Have I     │               │
│    │                     │ seen abc123?      │               │
│    │                     └─────────┬─────────┘               │
│    │                               │                         │
│    │                        NO     │                         │
│    │                     ┌─────────┴─────────┐               │
│    │                     │ 1. Store abc123   │               │
│    │                     │ 2. Process payment│               │
│    │                     │ 3. Store result   │               │
│    │                     └─────────┬─────────┘               │
│    │                               │                         │
│    │  200 OK                       │                         │
│    │  {status: "completed"}        │                         │
│    │ <─────────────────────────────│                         │
│    │                               │                         │
│    │  [Network timeout, client retries]                      │
│    │                               │                         │
│    │  POST /payment                │                         │
│    │  Idempotency-Key: abc123      │                         │
│    │  {amount: $100}               │                         │
│    │ ─────────────────────────────>│                         │
│    │                               │                         │
│    │                     ┌─────────┴─────────┐               │
│    │                     │ Check: Have I     │               │
│    │                     │ seen abc123?      │               │
│    │                     └─────────┬─────────┘               │
│    │                               │                         │
│    │                        YES    │                         │
│    │                     ┌─────────┴─────────┐               │
│    │                     │ Return stored     │               │
│    │                     │ result (no retry) │               │
│    │                     └─────────┬─────────┘               │
│    │                               │                         │
│    │  200 OK                       │                         │
│    │  {status: "completed"}        │                         │
│    │ <─────────────────────────────│                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Storage for Idempotency Keys

Where do you store the keys and results?

**Option 1: In-Memory Cache (Redis)**
```
Pros: Fast, simple
Cons: Lost on restart, limited by memory
TTL: 24-48 hours typically
```

**Option 2: Database Table**
```
Pros: Durable, queryable
Cons: Slower, adds DB load
TTL: Use scheduled cleanup
```

**Option 3: Hybrid**
```
Check Redis first (fast path)
Fall back to DB (durability)
Write to both on new requests
```

### The Idempotency Key Lifecycle

```
1. CLIENT GENERATES KEY
   - UUID: "550e8400-e29b-41d4-a716-446655440000"
   - Or: Hash of request content
   - Or: Business identifier (order_id + action)

2. SERVER RECEIVES REQUEST
   - Extract idempotency key from header
   - Check if key exists in store

3. KEY NOT FOUND (First request)
   - Mark key as "processing" (prevents race conditions)
   - Execute the operation
   - Store result with key
   - Mark key as "completed"
   - Return result

4. KEY FOUND (Duplicate request)
   - If status = "processing": Return 409 Conflict (or wait)
   - If status = "completed": Return stored result
   - If status = "failed": Allow retry (optional)

5. KEY EXPIRATION
   - Keys expire after TTL (e.g., 24 hours)
   - After expiration, same key = new request
```

---

## 4️⃣ Simulation-First Explanation

Let's trace through a real scenario step by step.

### Scenario: Processing a Payment

**Setup:**
- User wants to pay $100
- Client generates idempotency key: `pay_abc123`
- Network is unreliable (50% packet loss)

### First Attempt

```
Time 0ms: Client sends request
  POST /api/payments
  Idempotency-Key: pay_abc123
  Body: {"amount": 100, "currency": "USD", "card": "****1234"}

Time 5ms: Server receives request
  - Checks Redis: GET idempotency:pay_abc123 → NULL (not found)
  - Sets lock: SET idempotency:pay_abc123 "processing" NX EX 300
  - Lock acquired successfully

Time 10ms: Server processes payment
  - Calls payment processor (Stripe, etc.)
  - Payment successful, transaction_id: "txn_xyz789"

Time 50ms: Server stores result
  - SET idempotency:pay_abc123 '{"status":"completed","txn":"txn_xyz789"}' EX 86400

Time 55ms: Server sends response
  - 200 OK {"status": "completed", "transaction_id": "txn_xyz789"}

Time 100ms: Response lost in network! Client never receives it.
```

### Client Retry (Same Idempotency Key)

```
Time 5000ms: Client timeout, retries with SAME key
  POST /api/payments
  Idempotency-Key: pay_abc123  ← Same key!
  Body: {"amount": 100, "currency": "USD", "card": "****1234"}

Time 5005ms: Server receives request
  - Checks Redis: GET idempotency:pay_abc123
  - Found! {"status":"completed","txn":"txn_xyz789"}

Time 5006ms: Server returns cached result (NO new payment!)
  - 200 OK {"status": "completed", "transaction_id": "txn_xyz789"}

Time 5050ms: Client receives response
  - Success! User charged exactly once.
```

### What If Client Used Different Key?

```
Time 5000ms: Client retries with NEW key (BAD!)
  POST /api/payments
  Idempotency-Key: pay_def456  ← Different key!
  Body: {"amount": 100, "currency": "USD", "card": "****1234"}

Time 5005ms: Server receives request
  - Checks Redis: GET idempotency:pay_def456 → NULL
  - New key! Processes payment AGAIN.

Time 5050ms: User charged TWICE! 💸💸
```

**Lesson**: The client MUST use the same idempotency key for retries.

---

## 5️⃣ How Engineers Actually Use This in Production

### Stripe's Implementation

Stripe is the gold standard for idempotency in payment APIs.

**Their approach:**
1. Idempotency key is a header: `Idempotency-Key: <key>`
2. Keys are scoped to API key (your key can't collide with others)
3. Keys expire after 24 hours
4. Same key + different request body = error (prevents misuse)

```bash
# Stripe API example
curl https://api.stripe.com/v1/charges \
  -u sk_test_xxx: \
  -H "Idempotency-Key: order_12345_charge" \
  -d amount=2000 \
  -d currency=usd \
  -d source=tok_visa
```

**Stripe's documentation says:**
> "Idempotency keys are sent in the `Idempotency-Key` header and uniquely identify a request. If Stripe receives a request with a key that has already been used, Stripe returns the response from the original request."

### Amazon's Order Processing

Amazon uses a different pattern: **business-level idempotency**.

Instead of generic keys, they use:
- Order ID + Action (e.g., `order_123_place`)
- Cart ID + Checkout attempt
- Request deduplication at multiple layers

### Netflix's Approach

Netflix deals with idempotency in their microservices:

1. **Zuul Gateway**: Deduplicates requests at the edge
2. **Service Layer**: Each service has its own idempotency handling
3. **Hystrix**: Circuit breakers prevent cascade of retries

### Uber's Request Deduplication

Uber uses idempotency for:
- Ride requests (prevent multiple drivers)
- Payment processing
- Driver earnings calculations

They combine:
- Client-generated request IDs
- Server-side deduplication
- Database constraints (unique indexes)

---

## 6️⃣ How to Implement or Apply It

### Complete Java Implementation

#### Maven Dependencies

```xml
<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Redis for idempotency storage -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    
    <!-- JSON processing -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```

#### Idempotency Key Storage

```java
package com.systemdesign.idempotency;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import java.time.Duration;
import java.util.Optional;

/**
 * Stores idempotency keys and their results in Redis.
 * Keys expire after 24 hours by default.
 */
@Component
public class IdempotencyKeyStore {
    
    private static final String KEY_PREFIX = "idempotency:";
    private static final Duration DEFAULT_TTL = Duration.ofHours(24);
    
    private final StringRedisTemplate redisTemplate;
    private final ObjectMapper objectMapper;
    
    public IdempotencyKeyStore(StringRedisTemplate redisTemplate, 
                                ObjectMapper objectMapper) {
        this.redisTemplate = redisTemplate;
        this.objectMapper = objectMapper;
    }
    
    /**
     * Attempts to acquire a lock for the given idempotency key.
     * Returns true if this is the first request with this key.
     * Returns false if the key already exists (duplicate request).
     */
    public boolean tryAcquire(String key) {
        String redisKey = KEY_PREFIX + key;
        // SET NX = Set if Not eXists
        // This is atomic - prevents race conditions
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(redisKey, "{\"status\":\"processing\"}", DEFAULT_TTL);
        return Boolean.TRUE.equals(acquired);
    }
    
    /**
     * Stores the result for an idempotency key.
     * Called after the operation completes successfully.
     */
    public void storeResult(String key, IdempotencyResult result) {
        String redisKey = KEY_PREFIX + key;
        try {
            String json = objectMapper.writeValueAsString(result);
            redisTemplate.opsForValue().set(redisKey, json, DEFAULT_TTL);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to serialize result", e);
        }
    }
    
    /**
     * Retrieves the stored result for an idempotency key.
     * Returns empty if the key doesn't exist or is still processing.
     */
    public Optional<IdempotencyResult> getResult(String key) {
        String redisKey = KEY_PREFIX + key;
        String json = redisTemplate.opsForValue().get(redisKey);
        
        if (json == null) {
            return Optional.empty();
        }
        
        try {
            IdempotencyResult result = objectMapper.readValue(json, IdempotencyResult.class);
            if ("processing".equals(result.getStatus())) {
                // Request is still being processed
                return Optional.empty();
            }
            return Optional.of(result);
        } catch (JsonProcessingException e) {
            return Optional.empty();
        }
    }
    
    /**
     * Marks a key as failed, allowing retry.
     */
    public void markFailed(String key, String errorMessage) {
        String redisKey = KEY_PREFIX + key;
        IdempotencyResult result = new IdempotencyResult();
        result.setStatus("failed");
        result.setErrorMessage(errorMessage);
        try {
            String json = objectMapper.writeValueAsString(result);
            redisTemplate.opsForValue().set(redisKey, json, Duration.ofMinutes(5));
        } catch (JsonProcessingException e) {
            // Delete the key to allow retry
            redisTemplate.delete(redisKey);
        }
    }
    
    /**
     * Removes an idempotency key (for cleanup or allowing retry).
     */
    public void remove(String key) {
        redisTemplate.delete(KEY_PREFIX + key);
    }
}
```

#### Result Model

```java
package com.systemdesign.idempotency;

import com.fasterxml.jackson.annotation.JsonInclude;

/**
 * Represents the stored result of an idempotent operation.
 */
@JsonInclude(JsonInclude.Include.NON_NULL)
public class IdempotencyResult {
    
    private String status;          // "processing", "completed", "failed"
    private Integer httpStatus;     // HTTP status code to return
    private Object responseBody;    // The response to return
    private String errorMessage;    // Error message if failed
    
    // Getters and setters
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    
    public Integer getHttpStatus() { return httpStatus; }
    public void setHttpStatus(Integer httpStatus) { this.httpStatus = httpStatus; }
    
    public Object getResponseBody() { return responseBody; }
    public void setResponseBody(Object responseBody) { this.responseBody = responseBody; }
    
    public String getErrorMessage() { return errorMessage; }
    public void setErrorMessage(String errorMessage) { this.errorMessage = errorMessage; }
    
    // Factory methods
    public static IdempotencyResult success(int httpStatus, Object body) {
        IdempotencyResult result = new IdempotencyResult();
        result.setStatus("completed");
        result.setHttpStatus(httpStatus);
        result.setResponseBody(body);
        return result;
    }
    
    public static IdempotencyResult failure(String error) {
        IdempotencyResult result = new IdempotencyResult();
        result.setStatus("failed");
        result.setErrorMessage(error);
        return result;
    }
}
```

#### Idempotency Filter (Interceptor)

```java
package com.systemdesign.idempotency;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;
import org.springframework.web.util.ContentCachingResponseWrapper;

import java.io.IOException;
import java.util.Optional;
import java.util.Set;

/**
 * Filter that handles idempotency for POST/PATCH requests.
 * Checks for Idempotency-Key header and deduplicates requests.
 */
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class IdempotencyFilter extends OncePerRequestFilter {
    
    private static final String IDEMPOTENCY_KEY_HEADER = "Idempotency-Key";
    private static final Set<String> IDEMPOTENT_METHODS = Set.of("POST", "PATCH");
    
    private final IdempotencyKeyStore keyStore;
    private final ObjectMapper objectMapper;
    
    public IdempotencyFilter(IdempotencyKeyStore keyStore, ObjectMapper objectMapper) {
        this.keyStore = keyStore;
        this.objectMapper = objectMapper;
    }
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) 
            throws ServletException, IOException {
        
        // Only apply to POST/PATCH requests
        if (!IDEMPOTENT_METHODS.contains(request.getMethod())) {
            filterChain.doFilter(request, response);
            return;
        }
        
        // Get idempotency key from header
        String idempotencyKey = request.getHeader(IDEMPOTENCY_KEY_HEADER);
        
        // If no key provided, proceed without idempotency
        if (idempotencyKey == null || idempotencyKey.isBlank()) {
            filterChain.doFilter(request, response);
            return;
        }
        
        // Check if we've seen this key before
        Optional<IdempotencyResult> existingResult = keyStore.getResult(idempotencyKey);
        
        if (existingResult.isPresent()) {
            // Duplicate request! Return cached response.
            IdempotencyResult cached = existingResult.get();
            response.setStatus(cached.getHttpStatus());
            response.setContentType("application/json");
            response.setHeader("X-Idempotency-Replayed", "true");
            objectMapper.writeValue(response.getOutputStream(), cached.getResponseBody());
            return;
        }
        
        // Try to acquire lock for this key
        if (!keyStore.tryAcquire(idempotencyKey)) {
            // Another request with this key is being processed
            response.setStatus(HttpServletResponse.SC_CONFLICT);
            response.setContentType("application/json");
            objectMapper.writeValue(response.getOutputStream(), 
                new ErrorResponse("Request with this idempotency key is already being processed"));
            return;
        }
        
        // Wrap response to capture the output
        ContentCachingResponseWrapper responseWrapper = 
            new ContentCachingResponseWrapper(response);
        
        try {
            // Process the actual request
            filterChain.doFilter(request, responseWrapper);
            
            // Store the result
            byte[] responseBody = responseWrapper.getContentAsByteArray();
            Object body = responseBody.length > 0 
                ? objectMapper.readValue(responseBody, Object.class) 
                : null;
            
            IdempotencyResult result = IdempotencyResult.success(
                responseWrapper.getStatus(), 
                body
            );
            keyStore.storeResult(idempotencyKey, result);
            
            // Copy the cached content to the actual response
            responseWrapper.copyBodyToResponse();
            
        } catch (Exception e) {
            // Mark as failed to allow retry
            keyStore.markFailed(idempotencyKey, e.getMessage());
            throw e;
        }
    }
    
    private record ErrorResponse(String error) {}
}
```

#### Payment Controller Example

```java
package com.systemdesign.idempotency;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Map;
import java.util.UUID;

/**
 * Example payment controller demonstrating idempotency.
 * The IdempotencyFilter handles deduplication automatically.
 */
@RestController
@RequestMapping("/api/payments")
public class PaymentController {
    
    private final PaymentService paymentService;
    
    public PaymentController(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
    
    /**
     * Create a new payment.
     * 
     * Idempotency is handled by the filter. If the same Idempotency-Key
     * is sent twice, the second request returns the cached response.
     */
    @PostMapping
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestBody PaymentRequest request,
            @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {
        
        // The filter has already checked for duplicates.
        // If we reach here, this is a new request.
        
        PaymentResult result = paymentService.processPayment(
            request.getAmount(),
            request.getCurrency(),
            request.getCardToken()
        );
        
        PaymentResponse response = new PaymentResponse(
            result.getTransactionId(),
            result.getStatus(),
            request.getAmount(),
            request.getCurrency()
        );
        
        return ResponseEntity.ok(response);
    }
    
    // Request/Response DTOs
    public record PaymentRequest(
        int amount,
        String currency,
        String cardToken
    ) {}
    
    public record PaymentResponse(
        String transactionId,
        String status,
        int amount,
        String currency
    ) {}
}
```

#### Client-Side Implementation

```java
package com.systemdesign.idempotency.client;

import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;

import java.util.UUID;

/**
 * Example client that properly uses idempotency keys.
 */
public class PaymentClient {
    
    private final RestTemplate restTemplate;
    private final String baseUrl;
    
    public PaymentClient(String baseUrl) {
        this.restTemplate = new RestTemplate();
        this.baseUrl = baseUrl;
    }
    
    /**
     * Creates a payment with automatic retry and idempotency.
     * 
     * Key insight: Generate the idempotency key ONCE before the first attempt,
     * then reuse it for all retries.
     */
    public PaymentResponse createPayment(int amount, String currency, String cardToken) {
        // Generate idempotency key ONCE
        String idempotencyKey = generateIdempotencyKey(amount, currency, cardToken);
        
        int maxRetries = 3;
        int attempt = 0;
        
        while (attempt < maxRetries) {
            attempt++;
            try {
                return doCreatePayment(amount, currency, cardToken, idempotencyKey);
            } catch (Exception e) {
                if (attempt >= maxRetries) {
                    throw new RuntimeException("Payment failed after " + maxRetries + " attempts", e);
                }
                // Wait before retry (exponential backoff)
                sleep(1000 * attempt);
            }
        }
        
        throw new RuntimeException("Should not reach here");
    }
    
    private PaymentResponse doCreatePayment(int amount, String currency, 
                                            String cardToken, String idempotencyKey) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("Idempotency-Key", idempotencyKey);  // Same key for all retries!
        
        PaymentRequest request = new PaymentRequest(amount, currency, cardToken);
        HttpEntity<PaymentRequest> entity = new HttpEntity<>(request, headers);
        
        ResponseEntity<PaymentResponse> response = restTemplate.exchange(
            baseUrl + "/api/payments",
            HttpMethod.POST,
            entity,
            PaymentResponse.class
        );
        
        // Check if this was a replayed response
        if ("true".equals(response.getHeaders().getFirst("X-Idempotency-Replayed"))) {
            System.out.println("Response was replayed from cache (duplicate request)");
        }
        
        return response.getBody();
    }
    
    /**
     * Generates a deterministic idempotency key based on request content.
     * This ensures the same logical request always uses the same key.
     */
    private String generateIdempotencyKey(int amount, String currency, String cardToken) {
        // Option 1: UUID (unique per call, must be stored/reused)
        // return UUID.randomUUID().toString();
        
        // Option 2: Hash of content (deterministic)
        String content = amount + ":" + currency + ":" + cardToken + ":" + System.currentTimeMillis() / 60000;
        return "pay_" + Integer.toHexString(content.hashCode());
    }
    
    private void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    // DTOs
    public record PaymentRequest(int amount, String currency, String cardToken) {}
    public record PaymentResponse(String transactionId, String status, int amount, String currency) {}
}
```

### Database-Level Idempotency

Sometimes you want idempotency at the database level:

```java
package com.systemdesign.idempotency;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

/**
 * Uses database constraints for idempotency.
 * The unique constraint prevents duplicate processing.
 */
@Repository
public class IdempotentOrderRepository {
    
    private final JdbcTemplate jdbcTemplate;
    
    public IdempotentOrderRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
    
    /**
     * Creates an order idempotently using database constraints.
     * 
     * The orders table has: UNIQUE INDEX (idempotency_key)
     * If we try to insert a duplicate, the database rejects it.
     */
    @Transactional
    public OrderResult createOrder(String idempotencyKey, OrderRequest request) {
        // First, try to find existing order with this key
        String selectSql = "SELECT id, status, total FROM orders WHERE idempotency_key = ?";
        var existing = jdbcTemplate.query(selectSql, 
            (rs, rowNum) -> new OrderResult(
                rs.getString("id"),
                rs.getString("status"),
                rs.getInt("total")
            ),
            idempotencyKey
        );
        
        if (!existing.isEmpty()) {
            // Already processed, return existing result
            return existing.get(0);
        }
        
        // Try to insert new order
        String orderId = generateOrderId();
        String insertSql = """
            INSERT INTO orders (id, idempotency_key, user_id, total, status, created_at)
            VALUES (?, ?, ?, ?, 'pending', NOW())
            ON DUPLICATE KEY UPDATE id = id
            """;
        
        int rowsAffected = jdbcTemplate.update(insertSql, 
            orderId, idempotencyKey, request.userId(), request.total());
        
        if (rowsAffected == 0) {
            // Duplicate key, fetch and return existing
            return jdbcTemplate.queryForObject(selectSql,
                (rs, rowNum) -> new OrderResult(
                    rs.getString("id"),
                    rs.getString("status"),
                    rs.getInt("total")
                ),
                idempotencyKey
            );
        }
        
        // New order created
        return new OrderResult(orderId, "pending", request.total());
    }
    
    private String generateOrderId() {
        return "ORD-" + System.currentTimeMillis();
    }
    
    public record OrderRequest(String userId, int total) {}
    public record OrderResult(String orderId, String status, int total) {}
}
```

**SQL Schema:**
```sql
CREATE TABLE orders (
    id VARCHAR(50) PRIMARY KEY,
    idempotency_key VARCHAR(100) NOT NULL,
    user_id VARCHAR(50) NOT NULL,
    total INT NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    
    UNIQUE INDEX idx_idempotency_key (idempotency_key)
);
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

#### 1. Client Generates New Key on Retry

**Wrong:**
```java
// BAD: New key on each attempt
for (int i = 0; i < 3; i++) {
    String key = UUID.randomUUID().toString();  // New key each time!
    try {
        return callApi(key, request);
    } catch (Exception e) {
        // Retry with new key = duplicate processing!
    }
}
```

**Right:**
```java
// GOOD: Same key for all retries
String key = UUID.randomUUID().toString();  // Generate once
for (int i = 0; i < 3; i++) {
    try {
        return callApi(key, request);  // Reuse same key
    } catch (Exception e) {
        // Retry with same key = safe
    }
}
```

#### 2. Ignoring the "Processing" State

**Wrong:**
```java
// BAD: No handling for concurrent requests
if (keyStore.exists(key)) {
    return keyStore.getResult(key);  // Might be "processing"!
}
keyStore.set(key, "processing");
// ... process ...
```

**Right:**
```java
// GOOD: Handle processing state
Optional<Result> existing = keyStore.getResult(key);
if (existing.isPresent()) {
    if (existing.get().isProcessing()) {
        throw new ConflictException("Request in progress");
    }
    return existing.get();
}
if (!keyStore.tryAcquire(key)) {  // Atomic check-and-set
    throw new ConflictException("Request in progress");
}
```

#### 3. Not Storing Failed Results

**Wrong:**
```java
try {
    result = processPayment();
    keyStore.store(key, result);
} catch (Exception e) {
    // Key is stuck in "processing" forever!
    throw e;
}
```

**Right:**
```java
try {
    result = processPayment();
    keyStore.store(key, result);
} catch (Exception e) {
    keyStore.markFailed(key);  // Allow retry
    throw e;
}
```

#### 4. Same Key for Different Operations

**Wrong:**
```java
// BAD: Same key for different operations
String key = "user_123";
createOrder(key, orderData);   // Creates order
updateOrder(key, updateData);  // Returns cached create response!
```

**Right:**
```java
// GOOD: Include operation type in key
String createKey = "user_123_create_order_" + timestamp;
String updateKey = "user_123_update_order_" + orderId;
```

### Performance Gotchas

#### Redis Latency

```
Every request now has:
1. Redis GET (check for existing key): ~1ms
2. Redis SET (acquire lock): ~1ms
3. Actual operation: Xms
4. Redis SET (store result): ~1ms

Total overhead: ~3ms per request

For high-throughput systems, consider:
- Local cache in front of Redis
- Batch operations
- Async result storage
```

#### Storage Growth

```
If you have:
- 10,000 requests/second
- Each result is 1KB
- 24-hour TTL

Storage needed:
- 10,000 × 86,400 × 1KB = 864 GB/day

Solutions:
- Shorter TTL for non-critical operations
- Compress stored results
- Store only essential fields
```

---

## 8️⃣ When NOT to Use This

### When Idempotency is Unnecessary

1. **GET requests**: Already idempotent by definition
2. **Read-only operations**: No state change, no risk
3. **Operations with natural idempotency**: 
   - `SET x = 5` (not `x = x + 1`)
   - `DELETE WHERE id = 123`

### When Idempotency is Overkill

1. **Internal service-to-service calls with reliable networking**
   - If you have synchronous calls within the same datacenter
   - Network failures are rare

2. **Batch processing with checkpoints**
   - If you can resume from a checkpoint
   - Idempotency per-item might be unnecessary

3. **Event sourcing systems**
   - Events are naturally deduplicated by sequence number
   - The event store handles idempotency

### Anti-Patterns

1. **Idempotency key in the request body**
   - Should be in header (body might not be parsed on retry)

2. **User-provided idempotency keys without validation**
   - Users might reuse keys incorrectly
   - Validate format and uniqueness

3. **Infinite TTL for idempotency keys**
   - Storage grows unbounded
   - Old keys become irrelevant

---

## 9️⃣ Comparison with Alternatives

### Idempotency vs Transactions

| Aspect | Idempotency | Transactions |
|--------|-------------|--------------|
| Scope | Single operation | Multiple operations |
| Purpose | Prevent duplicates | Ensure atomicity |
| Failure handling | Retry safely | Rollback on failure |
| Complexity | Low | High |
| Use together? | Yes! They complement each other |

### Idempotency vs Exactly-Once Delivery

| Aspect | Idempotency | Exactly-Once |
|--------|-------------|--------------|
| Guarantee | Same result on retry | Message processed once |
| Implementation | Application layer | Message broker |
| Examples | Stripe, REST APIs | Kafka transactions |
| Limitation | Requires client cooperation | Broker-specific |

### Idempotency Key Strategies

| Strategy | Pros | Cons | Use When |
|----------|------|------|----------|
| UUID | Simple, unique | Must store/reuse | General purpose |
| Hash of content | Deterministic | Collisions possible | Same content = same op |
| Business ID | Meaningful | Must be unique | Order ID, user action |
| Timestamp-based | Time-scoped | Clock sync issues | Time-sensitive ops |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q1: What is idempotency and why does it matter?**

**Answer:**
Idempotency means that performing an operation multiple times has the same effect as performing it once. It matters because in distributed systems, network failures and retries are common. Without idempotency, a retry might cause duplicate actions, like charging a customer twice.

For example, if a payment request times out and the client retries, an idempotent system will recognize the duplicate and return the original result instead of processing a second payment.

**Q2: Which HTTP methods are idempotent?**

**Answer:**
- **GET, HEAD, OPTIONS**: Always idempotent (read-only)
- **PUT**: Idempotent because it sets a resource to a specific state. Calling `PUT /user/123 {name: "John"}` multiple times always results in the same state.
- **DELETE**: Idempotent because deleting something that's already deleted is a no-op.
- **POST**: NOT idempotent by default. Each POST typically creates a new resource.
- **PATCH**: Not guaranteed idempotent. Depends on the operation (increment vs set).

### L5 (Senior) Questions

**Q3: How would you implement idempotency for a payment system?**

**Answer:**
I would use an idempotency key approach:

1. **Client generates a unique key** (UUID or hash of request) before the first attempt and includes it in the request header.

2. **Server checks Redis** for the key:
   - Not found: Acquire lock, process payment, store result
   - Found with "processing": Return 409 Conflict
   - Found with result: Return cached result

3. **Atomic lock acquisition** using Redis `SET NX` (set if not exists) to prevent race conditions.

4. **Store the complete response** including status code and body, so retries get identical responses.

5. **TTL of 24-48 hours** for keys. After expiration, same key is treated as new.

6. **Handle failures**: If processing fails, mark the key as failed to allow retry, or delete the key.

**Q4: How do you handle the case where the operation succeeds but storing the result fails?**

**Answer:**
This is a tricky edge case. Options:

1. **Wrap in a transaction** (if possible): Store the result in the same transaction as the operation. Both succeed or both fail.

2. **Two-phase approach**: 
   - First, store "processing" status
   - Then, perform operation
   - Finally, update to "completed" with result
   - If the final update fails, the next retry will see "processing" and can check the actual state

3. **Operation-specific recovery**:
   - For payments: Check with payment processor if transaction exists
   - For orders: Query the database for the order
   - Return the actual state, update the idempotency store

4. **Accept the duplicate risk**: For non-critical operations, it might be acceptable to occasionally process twice.

### L6 (Staff) Questions

**Q5: Design an idempotency system that works across multiple datacenters.**

**Answer:**
Cross-datacenter idempotency is challenging because of replication lag. Here's my approach:

**Option 1: Route by idempotency key**
- Hash the idempotency key to determine which datacenter "owns" it
- All requests with that key go to the same datacenter
- Simple but adds latency for some requests

**Option 2: Synchronous cross-DC check**
- Before processing, check all datacenters for the key
- High latency but consistent
- Use for critical operations (payments)

**Option 3: Optimistic with reconciliation**
- Process locally, replicate asynchronously
- If duplicate detected later, compensate (refund, etc.)
- Good for operations where duplicates are recoverable

**Option 4: Global consensus (Spanner-like)**
- Use a globally consistent store for idempotency keys
- Highest consistency, highest latency
- For financial systems

For most systems, I'd recommend Option 1 (route by key) for simplicity, with Option 2 as a fallback for critical operations.

**Q6: How would you test an idempotency implementation?**

**Answer:**
I would test at multiple levels:

**Unit tests:**
- Key store correctly identifies duplicates
- Lock acquisition is atomic
- Result storage and retrieval works
- TTL expiration works

**Integration tests:**
- Concurrent requests with same key (only one processes)
- Retry after timeout returns cached result
- Different keys process independently
- Failed operations allow retry

**Chaos tests:**
- Kill the service mid-processing, verify retry works
- Network partition between service and Redis
- Redis failover during request
- Clock skew between servers

**Load tests:**
- High concurrency with many duplicate keys
- Memory/storage growth over time
- Latency impact of idempotency checks

**Production monitoring:**
- Track duplicate request rate
- Alert on high conflict rate (might indicate client bug)
- Monitor Redis memory usage

---

## 1️⃣1️⃣ One Clean Mental Summary

Idempotency ensures that performing an operation multiple times has the same effect as performing it once. This is critical in distributed systems where network failures cause retries. The standard implementation uses an idempotency key: the client generates a unique key for each logical operation and sends it with every attempt. The server stores this key with the result after processing. On retry, the server recognizes the key and returns the cached result instead of processing again. This transforms "at-least-once" delivery into "exactly-once" semantics. Key implementation details: use atomic lock acquisition (Redis SET NX), store complete responses, set reasonable TTLs (24 hours), and handle the "processing" state for concurrent requests. The client must reuse the same key for retries, never generate a new key.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│                  IDEMPOTENCY CHEAT SHEET                     │
├─────────────────────────────────────────────────────────────┤
│ DEFINITION                                                   │
│   f(f(x)) = f(x)                                            │
│   Multiple executions = Same result as one execution        │
├─────────────────────────────────────────────────────────────┤
│ HTTP METHODS                                                 │
│   Idempotent: GET, PUT, DELETE, HEAD, OPTIONS               │
│   NOT Idempotent: POST, PATCH (usually)                     │
├─────────────────────────────────────────────────────────────┤
│ IMPLEMENTATION PATTERN                                       │
│   1. Client generates unique key (once, before first try)   │
│   2. Client sends key in header with every attempt          │
│   3. Server checks: key exists?                             │
│      - No: Acquire lock, process, store result              │
│      - Yes (processing): Return 409 Conflict                │
│      - Yes (completed): Return cached result                │
├─────────────────────────────────────────────────────────────┤
│ KEY GENERATION                                               │
│   UUID: Simple, must reuse for retries                      │
│   Hash: Deterministic, same content = same key              │
│   Business ID: order_123_place, user_456_update             │
├─────────────────────────────────────────────────────────────┤
│ STORAGE                                                      │
│   Redis: Fast, use SET NX for atomic lock                   │
│   Database: Durable, use UNIQUE constraint                  │
│   TTL: 24-48 hours typical                                  │
├─────────────────────────────────────────────────────────────┤
│ COMMON MISTAKES                                              │
│   ✗ New key on each retry                                   │
│   ✗ Ignoring "processing" state                             │
│   ✗ Not handling failures                                   │
│   ✗ Same key for different operations                       │
├─────────────────────────────────────────────────────────────┤
│ FORMULA                                                      │
│   At-least-once delivery + Idempotent ops = Exactly-once    │
└─────────────────────────────────────────────────────────────┘
```

