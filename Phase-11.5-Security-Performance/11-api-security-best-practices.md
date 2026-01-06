# 🔐 API Security Best Practices: Protecting Your API Endpoints

---

## 0️⃣ Prerequisites

Before diving into API security, you should understand:

- **REST APIs**: How HTTP-based APIs work (GET, POST, PUT, DELETE). Covered in Phase 2.
- **Authentication**: How to verify user identity (JWT, OAuth2). Covered in `03-authentication-jwt.md` and `04-oauth2-openid-connect.md`.
- **Authorization**: How to control access to resources. Covered in `05-authorization.md`.
- **HTTPS/TLS**: How data is encrypted in transit. Covered in `01-https-tls.md`.
- **Rate Limiting**: How to limit request frequency. Covered in Phase 5.

**Quick Refresher**: API security ensures that only authorized users can access your endpoints, data is protected in transit, and the API is resilient to attacks.

---

## 1️⃣ What Problem Does API Security Exist to Solve?

### The Core Problem: APIs Are Attack Vectors

APIs expose your application's functionality to the internet:
- **Public APIs**: Anyone can call them
- **Internal APIs**: Should only be accessible to authorized services
- **Partner APIs**: Limited access for specific partners

### What Systems Looked Like Before API Security

**The Dark Ages (No Security)**:

```java
// ❌ TERRIBLE: No authentication, no authorization
@RestController
public class UserController {
    
    @GetMapping("/api/users/{id}")
    public User getUser(@PathVariable Long id) {
        // Anyone can call this!
        // No authentication check
        // No authorization check
        return userRepository.findById(id).orElseThrow();
    }
    
    @DeleteMapping("/api/users/{id}")
    public void deleteUser(@PathVariable Long id) {
        // Anyone can delete any user!
        userRepository.deleteById(id);
    }
}
```

**Problems with this approach**:
1. Anyone can access any endpoint
2. No way to track who made requests
3. No rate limiting (DDoS vulnerable)
4. No input validation (injection attacks)
5. Sensitive data exposed in responses

### Real-World API Security Breaches

**2018 Facebook API Breach**: 50 million users affected
- Access tokens exposed, attackers could access user accounts
- No proper token validation
- Impact: Account takeovers, data exposure

**2019 Capital One API Breach**: 100 million customers affected
- Misconfigured API endpoint allowed unauthorized access
- No proper authorization checks
- Impact: SSNs, credit scores, bank account numbers exposed

**2020 Twitter API Breach**: High-profile accounts compromised
- API keys leaked, attackers posted as verified users
- No proper key rotation
- Impact: Fake tweets from verified accounts

### What Breaks Without API Security

| Attack Type | Impact | Without Protection | With Protection |
|------------|--------|-------------------|-----------------|
| **Unauthorized Access** | Attacker accesses data | Any endpoint accessible | Authentication required |
| **Data Exposure** | Sensitive data leaked | All data in responses | Filtered, masked responses |
| **Injection Attacks** | SQL/NoSQL injection | Database compromised | Input validation, parameterized queries |
| **DDoS** | Service unavailable | No rate limiting | Rate limiting, throttling |
| **Man-in-the-Middle** | Data intercepted | Plain HTTP | HTTPS/TLS required |
| **Broken Authentication** | Account takeover | Weak tokens, no expiration | Strong tokens, refresh tokens |

---

## 2️⃣ Intuition and Mental Model

### The Building Security Analogy

Think of API security like a secure office building:

**Without Security (Open Building)**:
- Anyone can walk in
- No ID check
- No access control
- No visitor logs
- Anyone can access any room

**With Security (Protected Building)**:
1. **Entrance Check (Authentication)**: Show ID badge to enter
2. **Access Control (Authorization)**: Badge determines which floors you can access
3. **Visitor Log (Audit Trail)**: Every entry/exit is logged
4. **Rate Limiting**: Too many visits trigger alerts
5. **Encrypted Communication**: All conversations are private (HTTPS)
6. **Input Validation**: Security checks bags at entrance

### The API Gateway Analogy

Think of API security like a border checkpoint:

```mermaid
flowchart LR
    REQUEST["Request"]
    CHECKPOINT["Border Checkpoint"]
    COUNTRY["Your Country"]
    
    AUTH["Authentication:<br/>Show your passport"]
    AUTHZ["Authorization:<br/>Do you have a visa?"]
    RATE["Rate Limiting:<br/>Only 10 visits per day"]
    VALID["Input Validation:<br/>No dangerous items"]
    ENCRYPT["Encryption:<br/>All communication encrypted"]
    
    REQUEST --> CHECKPOINT
    CHECKPOINT --> AUTH
    CHECKPOINT --> AUTHZ
    CHECKPOINT --> RATE
    CHECKPOINT --> VALID
    CHECKPOINT --> ENCRYPT
    AUTH --> COUNTRY
    AUTHZ --> COUNTRY
    RATE --> COUNTRY
    VALID --> COUNTRY
    ENCRYPT --> COUNTRY
    
    style REQUEST fill:#e3f2fd
    style CHECKPOINT fill:#fff9c4
    style COUNTRY fill:#c8e6c9
    style AUTH fill:#fff9c4
    style AUTHZ fill:#fff9c4
    style RATE fill:#fce4ec
    style VALID fill:#c8e6c9
    style ENCRYPT fill:#c8e6c9
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
Request → [Border Checkpoint] → [Your Country]
           │
           ├─ Authentication: "Show your passport"
           ├─ Authorization: "Do you have a visa?"
           ├─ Rate Limiting: "Only 10 visits per day"
           ├─ Input Validation: "No dangerous items"
           └─ Encryption: "All communication encrypted"
```

</details>

Only authorized, validated requests pass through.

---

## 3️⃣ How API Security Works Internally

### Security Layers

```mermaid
flowchart TD
    REQUEST["Request"]
    
    LAYER1["1. Transport Security (HTTPS/TLS)<br/>- Encrypt all traffic<br/>- Certificate validation"]
    LAYER2["2. Authentication<br/>- Verify identity (JWT, API key)<br/>- Token validation"]
    LAYER3["3. Authorization<br/>- Check permissions (RBAC/ABAC)<br/>- Resource-level access control"]
    LAYER4["4. Input Validation<br/>- Sanitize input<br/>- Validate format"]
    LAYER5["5. Rate Limiting<br/>- Limit requests per time<br/>- Prevent abuse"]
    LAYER6["6. Business Logic<br/>- Process request"]
    LAYER7["7. Response Filtering<br/>- Remove sensitive data<br/>- Mask PII"]
    
    RESPONSE["Response"]
    
    REQUEST --> LAYER1
    LAYER1 --> LAYER2
    LAYER2 --> LAYER3
    LAYER3 --> LAYER4
    LAYER4 --> LAYER5
    LAYER5 --> LAYER6
    LAYER6 --> LAYER7
    LAYER7 --> RESPONSE
    
    style REQUEST fill:#e3f2fd
    style LAYER1 fill:#fff9c4
    style LAYER2 fill:#fff9c4
    style LAYER3 fill:#c8e6c9
    style LAYER4 fill:#c8e6c9
    style LAYER5 fill:#fce4ec
    style LAYER6 fill:#fff9c4
    style LAYER7 fill:#c8e6c9
    style RESPONSE fill:#e3f2fd
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────┐
│                    API SECURITY LAYERS                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Request                                                     │
│     │                                                        │
│     ▼                                                        │
│  ┌──────────────────────────────────────┐                  │
│  │  1. Transport Security (HTTPS/TLS)   │                  │
│  │     - Encrypt all traffic             │                  │
│  │     - Certificate validation          │                  │
│  └──────────────────────────────────────┘                  │
│     │                                                        │
│     ▼                                                        │
│  ┌──────────────────────────────────────┐                  │
│  │  2. Authentication                    │                  │
│  │     - Verify identity (JWT, API key)   │                  │
│  │     - Token validation                │                  │
│  └──────────────────────────────────────┘                  │
│     │                                                        │
│     ▼                                                        │
│  ┌──────────────────────────────────────┐                  │
│  │  3. Authorization                     │                  │
│  │     - Check permissions (RBAC/ABAC)   │                  │
│  │     - Resource-level access control   │                  │
│  └──────────────────────────────────────┘                  │
│     │                                                        │
│     ▼                                                        │
│  ┌──────────────────────────────────────┐                  │
│  │  4. Input Validation                  │                  │
│  │     - Sanitize input                  │                  │
│  │     - Validate format                │                  │
│  └──────────────────────────────────────┘                  │
│     │                                                        │
│     ▼                                                        │
│  ┌──────────────────────────────────────┐                  │
│  │  5. Rate Limiting                     │                  │
│  │     - Limit requests per time        │                  │
│  │     - Prevent abuse                   │                  │
│  └──────────────────────────────────────┘                  │
│     │                                                        │
│     ▼                                                        │
│  ┌──────────────────────────────────────┐                  │
│  │  6. Business Logic                    │                  │
│  │     - Process request                 │                  │
│  └──────────────────────────────────────┘                  │
│     │                                                        │
│     ▼                                                        │
│  ┌──────────────────────────────────────┐                  │
│  │  7. Response Filtering                │                  │
│  │     - Remove sensitive data           │                  │
│  │     - Mask PII                        │                  │
│  └──────────────────────────────────────┘                  │
│     │                                                        │
│     ▼                                                        │
│  Response                                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

</details>

### Request Flow with Security

**Step 1: Client Makes Request**
```http
GET /api/users/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
X-API-Key: sk_live_abc123xyz
```

**Step 2: API Gateway Validates**
1. Check HTTPS (reject HTTP)
2. Validate JWT token signature
3. Check token expiration
4. Verify API key (if required)
5. Check rate limits

**Step 3: Application Layer Security**
1. Extract user from JWT
2. Check authorization (can this user access user 123?)
3. Validate input (is 123 a valid user ID?)
4. Process request

**Step 4: Response Security**
1. Filter sensitive fields
2. Mask PII
3. Add security headers
4. Return response

---

## 4️⃣ Simulation-First Explanation

### Scenario: Protected User API

Let's trace a request to get user information:

**Step 1: Client Authenticates**
```http
POST /api/auth/login HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "email": "alice@example.com",
  "password": "secret123"
}
```

**Step 2: Server Validates and Returns Token**
```java
@PostMapping("/api/auth/login")
public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest request) {
    // 1. Validate credentials
    User user = userService.authenticate(request.getEmail(), request.getPassword());
    
    // 2. Generate JWT token
    String token = jwtService.generateToken(user);
    
    // 3. Return token
    return ResponseEntity.ok(new AuthResponse(token));
}
```

**Response**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 3600
}
```

**Step 3: Client Makes Authenticated Request**
```http
GET /api/users/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Step 4: Security Filter Intercepts**
```java
@Component
public class SecurityFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        // 1. Extract token
        String token = extractToken(httpRequest);
        
        // 2. Validate token
        if (!jwtService.validateToken(token)) {
            sendError(response, 401, "Invalid token");
            return;
        }
        
        // 3. Extract user from token
        User user = jwtService.getUserFromToken(token);
        
        // 4. Set user in request context
        SecurityContextHolder.getContext().setAuthentication(
            new JwtAuthentication(user)
        );
        
        // 5. Continue to controller
        chain.doFilter(request, response);
    }
}
```

**Step 5: Controller with Authorization**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        // 1. Get current user from security context
        User currentUser = getCurrentUser();
        
        // 2. Check authorization
        if (!currentUser.isAdmin() && !currentUser.getId().equals(id)) {
            throw new ForbiddenException("Cannot access other users' data");
        }
        
        // 3. Retrieve user
        User user = userRepository.findById(id).orElseThrow();
        
        // 4. Filter sensitive data
        UserResponse response = userMapper.toResponse(user);
        response.setSsn(null);  // Never return SSN
        response.setCreditCard(maskCreditCard(user.getCreditCard()));
        
        return ResponseEntity.ok(response);
    }
}
```

**Step 6: Response with Security Headers**
```java
@RestControllerAdvice
public class SecurityHeadersAdvice implements ResponseBodyAdvice<Object> {
    
    @Override
    public boolean supports(MethodParameter returnType, 
                           Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }
    
    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType,
                                 MediaType selectedContentType,
                                 Class<? extends HttpMessageConverter<?>> selectedConverterType,
                                 ServerHttpRequest request, ServerHttpResponse response) {
        // Add security headers
        response.getHeaders().add("X-Content-Type-Options", "nosniff");
        response.getHeaders().add("X-Frame-Options", "DENY");
        response.getHeaders().add("X-XSS-Protection", "1; mode=block");
        response.getHeaders().add("Strict-Transport-Security", 
            "max-age=31536000; includeSubDomains");
        
        return body;
    }
}
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Real-World Implementations

**Stripe API**:
- API key authentication (publishable + secret keys)
- Request signing for webhooks
- Rate limiting per API key
- Idempotency keys for safe retries
- Reference: Stripe API Documentation

**GitHub API**:
- OAuth2 authentication
- Personal access tokens
- Fine-grained permissions (scopes)
- Rate limiting with clear headers
- Reference: GitHub API Documentation

**AWS API**:
- Signature Version 4 for request signing
- IAM-based authorization
- Regional endpoints
- Request/response encryption

### Common Patterns

**Pattern 1: API Key Management**
```java
@Entity
public class ApiKey {
    @Id
    private String key;  // Hashed key
    
    private String userId;
    private List<String> scopes;  // ["read:users", "write:orders"]
    private LocalDateTime expiresAt;
    private boolean revoked;
    
    public boolean hasScope(String scope) {
        return scopes.contains(scope);
    }
}
```

**Pattern 2: Request Signing**
```java
public class RequestSigningService {
    
    public String signRequest(String method, String path, String body, 
                            String timestamp, String secretKey) {
        // Create signature string
        String signatureString = method + "\n" +
            path + "\n" +
            body + "\n" +
            timestamp;
        
        // Sign with HMAC-SHA256
        Mac mac = Mac.getInstance("HmacSHA256");
        SecretKeySpec keySpec = new SecretKeySpec(secretKey.getBytes(), "HmacSHA256");
        mac.init(keySpec);
        byte[] signature = mac.doFinal(signatureString.getBytes());
        
        return Base64.getEncoder().encodeToString(signature);
    }
    
    public boolean verifySignature(String signature, String method, String path,
                                  String body, String timestamp, String secretKey) {
        String expectedSignature = signRequest(method, path, body, timestamp, secretKey);
        return MessageDigest.isEqual(
            signature.getBytes(),
            expectedSignature.getBytes()
        );
    }
}
```

**Pattern 3: OAuth Scopes**
```java
@PreAuthorize("hasAuthority('read:users')")
@GetMapping("/api/users")
public List<User> getUsers() {
    // Only accessible if token has "read:users" scope
    return userService.getAllUsers();
}

@PreAuthorize("hasAuthority('write:users')")
@PostMapping("/api/users")
public User createUser(@RequestBody CreateUserRequest request) {
    // Only accessible if token has "write:users" scope
    return userService.createUser(request);
}
```

**Pattern 4: Idempotency Keys**
```java
@PostMapping("/api/orders")
public ResponseEntity<Order> createOrder(@RequestBody CreateOrderRequest request,
                                        @RequestHeader("Idempotency-Key") String idempotencyKey) {
    // Check if request already processed
    Optional<Order> existing = orderRepository.findByIdempotencyKey(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.ok(existing.get());  // Return existing order
    }
    
    // Process new order
    Order order = orderService.createOrder(request);
    order.setIdempotencyKey(idempotencyKey);
    orderRepository.save(order);
    
    return ResponseEntity.ok(order);
}
```

---

## 6️⃣ How to Implement or Apply It

### Spring Security Configuration

**Maven Dependencies**:
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.3</version>
    </dependency>
</dependencies>
```

**Security Configuration**:
```java
package com.example.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    private final JwtAuthenticationFilter jwtFilter;
    
    public SecurityConfig(JwtAuthenticationFilter jwtFilter) {
        this.jwtFilter = jwtFilter;
    }
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())  // Disable for stateless APIs
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/users/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .headers(headers -> headers
                .contentTypeOptions(contentTypeOptions -> {})
                .frameOptions(frameOptions -> frameOptions.deny())
                .xssProtection(xssProtection -> {})
            );
        
        return http.build();
    }
}
```

**JWT Authentication Filter**:
```java
package com.example.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.Collections;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtService jwtService;
    
    public JwtAuthenticationFilter(JwtService jwtService) {
        this.jwtService = jwtService;
    }
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                   HttpServletResponse response,
                                   FilterChain filterChain) throws ServletException, IOException {
        // 1. Extract token from header
        String token = extractToken(request);
        
        if (token != null && jwtService.validateToken(token)) {
            // 2. Extract user from token
            String username = jwtService.getUsernameFromToken(token);
            
            // 3. Create authentication object
            UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(
                    username,
                    null,
                    Collections.emptyList()  // Authorities loaded separately
                );
            authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            
            // 4. Set authentication in security context
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }
        
        // 5. Continue filter chain
        filterChain.doFilter(request, response);
    }
    
    private String extractToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

**Input Validation**:
```java
package com.example.validation;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public class CreateUserRequest {
    
    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;
    
    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 100, message = "Password must be 8-100 characters")
    private String password;
    
    // Getters and setters
}
```

**Controller with Validation**:
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @PostMapping
    public ResponseEntity<User> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        // @Valid ensures validation runs
        // If validation fails, returns 400 Bad Request
        
        User user = userService.createUser(request);
        return ResponseEntity.ok(user);
    }
}
```

**Rate Limiting with Redis**:
```java
package com.example.ratelimit;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

@Service
public class RateLimitService {
    
    private final RedisTemplate<String, String> redisTemplate;
    private static final int MAX_REQUESTS = 100;
    private static final Duration WINDOW = Duration.ofMinutes(1);
    
    public boolean isAllowed(String apiKey) {
        String key = "ratelimit:" + apiKey;
        
        // Increment counter
        Long count = redisTemplate.opsForValue().increment(key);
        
        // Set expiration on first request
        if (count == 1) {
            redisTemplate.expire(key, WINDOW.toSeconds(), TimeUnit.SECONDS);
        }
        
        // Check if limit exceeded
        return count <= MAX_REQUESTS;
    }
}
```

**Rate Limiting Interceptor**:
```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {
    
    private final RateLimitService rateLimitService;
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response,
                            Object handler) throws Exception {
        String apiKey = extractApiKey(request);
        
        if (!rateLimitService.isAllowed(apiKey)) {
            response.setStatus(429);  // Too Many Requests
            response.setHeader("X-RateLimit-Limit", "100");
            response.setHeader("X-RateLimit-Remaining", "0");
            response.setHeader("Retry-After", "60");
            return false;
        }
        
        return true;
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Tradeoffs

| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| **JWT Tokens** | Stateless, scalable | Can't revoke easily | Microservices, stateless APIs |
| **Session Tokens** | Can revoke easily | Requires session storage | Traditional web apps |
| **API Keys** | Simple, easy to implement | Less secure, no expiration | Internal services, partner APIs |
| **OAuth2** | Industry standard, flexible | Complex setup | Public APIs, third-party integration |

### Common Pitfalls

**Pitfall 1: Exposing Sensitive Data in URLs**
```java
// ❌ BAD: Sensitive data in URL
GET /api/users?apiKey=sk_live_abc123

// ✅ GOOD: API key in header
GET /api/users
Authorization: Bearer sk_live_abc123
```

**Pitfall 2: Not Validating Input**
```java
// ❌ BAD: No validation
@GetMapping("/api/users/{id}")
public User getUser(@PathVariable String id) {
    // SQL injection risk if id used in query
    return userRepository.findByCustomQuery(id);
}

// ✅ GOOD: Validate input
@GetMapping("/api/users/{id}")
public User getUser(@PathVariable @Min(1) Long id) {
    // Type-safe, validated
    return userRepository.findById(id).orElseThrow();
}
```

**Pitfall 3: Weak Token Expiration**
```java
// ❌ BAD: Tokens never expire
String token = jwtService.generateToken(user, Duration.ofDays(365));

// ✅ GOOD: Short-lived tokens with refresh
String token = jwtService.generateToken(user, Duration.ofMinutes(15));
String refreshToken = jwtService.generateRefreshToken(user, Duration.ofDays(30));
```

**Pitfall 4: Not Using HTTPS**
```java
// ❌ BAD: HTTP endpoint
@GetMapping("/api/users")
// No HTTPS enforcement

// ✅ GOOD: HTTPS only
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) {
    http.requiresChannel(channel -> 
        channel.anyRequest().requiresSecure());  // HTTPS only
    return http.build();
}
```

**Pitfall 5: Exposing Stack Traces**
```java
// ❌ BAD: Expose stack traces
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<?> handleException(Exception e) {
        return ResponseEntity.status(500)
            .body(e.getMessage() + "\n" + 
                  Arrays.toString(e.getStackTrace()));  // Exposes internals
    }
}

// ✅ GOOD: Generic error messages
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception e) {
        logger.error("Internal error", e);  // Log details server-side
        return ResponseEntity.status(500)
            .body(new ErrorResponse("Internal server error"));  // Generic message
    }
}
```

---

## 8️⃣ When NOT to Use This

### Anti-Patterns

**Don't Over-Secure**:
- Public read-only endpoints (product catalog) might not need authentication
- Internal microservices might use simpler authentication (mTLS)
- Development environments might use relaxed security

**When Simpler Approaches Work**:
- **Internal APIs**: Use mTLS instead of JWT
- **Read-only public data**: No authentication needed
- **Webhooks**: Use request signing instead of OAuth2

**Security vs Usability Tradeoff**:
- Too many security layers hurt developer experience
- Balance security with ease of use
- Document security requirements clearly

---

## 9️⃣ Comparison with Alternatives

### JWT vs Session Tokens

| Feature | JWT | Session Tokens |
|---------|-----|----------------|
| **Stateless** | ✅ Yes | ❌ No (requires storage) |
| **Revocable** | ❌ Hard to revoke | ✅ Easy to revoke |
| **Scalability** | ✅ Works across servers | ⚠️ Requires shared storage |
| **Size** | ⚠️ Larger (includes claims) | ✅ Small (just ID) |
| **Use Case** | Microservices, stateless | Traditional web apps |

**When to Choose Each**:
- **JWT**: Microservices, distributed systems, stateless APIs
- **Session Tokens**: Traditional web apps, need easy revocation

### API Keys vs OAuth2

| Feature | API Keys | OAuth2 |
|---------|----------|--------|
| **Complexity** | ✅ Simple | ⚠️ Complex |
| **Security** | ⚠️ Less secure | ✅ More secure |
| **Revocation** | ✅ Easy | ✅ Easy |
| **Scopes** | ❌ No | ✅ Yes |
| **Use Case** | Internal services, partners | Public APIs, third-party |

**When to Choose Each**:
- **API Keys**: Internal services, partner integrations, simple use cases
- **OAuth2**: Public APIs, third-party integrations, need fine-grained permissions

---

## 🔟 Interview Follow-up Questions WITH Answers

### Question 1: "How do you handle API versioning securely?"

**Answer**:
Multiple versioning strategies:

1. **URL Versioning**: `/api/v1/users`, `/api/v2/users`
2. **Header Versioning**: `Accept: application/vnd.api.v1+json`
3. **Query Parameter**: `/api/users?version=1`

**Security Considerations**:
- Deprecate old versions with clear timelines
- Maintain security patches for all supported versions
- Use version-specific authentication if needed

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    // Version 1 implementation
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    // Version 2 implementation with improved security
}
```

### Question 2: "How do you prevent API key leakage?"

**Answer**:
Multiple protection layers:

1. **Never in URLs**: Always use headers
2. **Client-side protection**: For web apps, use backend proxy
3. **Key rotation**: Regular rotation, invalidate leaked keys
4. **Monitoring**: Alert on unusual usage patterns
5. **Scope limiting**: Keys with minimal required permissions

```java
// Monitor for key leakage
public void detectKeyLeakage(String apiKey) {
    // Check for key in logs, error messages, public repos
    if (isKeyInPublicRepo(apiKey)) {
        alertSecurityTeam(apiKey);
        revokeKey(apiKey);
    }
}
```

### Question 3: "How do you implement request signing for webhooks?"

**Answer**:
HMAC-based request signing:

1. **Server signs requests**: Calculate HMAC-SHA256 of request body
2. **Client verifies signature**: Recalculate and compare
3. **Include timestamp**: Prevent replay attacks
4. **Use secret key**: Shared secret between server and client

```java
public class WebhookSigningService {
    
    public String signRequest(String body, String secret) {
        Mac mac = Mac.getInstance("HmacSHA256");
        SecretKeySpec keySpec = new SecretKeySpec(secret.getBytes(), "HmacSHA256");
        mac.init(keySpec);
        byte[] signature = mac.doFinal(body.getBytes());
        return "sha256=" + Hex.encodeHexString(signature);
    }
    
    public boolean verifySignature(String signature, String body, String secret) {
        String expected = signRequest(body, secret);
        return MessageDigest.isEqual(signature.getBytes(), expected.getBytes());
    }
}
```

### Question 4: "How do you handle CORS securely for APIs?"

**Answer**:
Configure CORS properly:

1. **Whitelist origins**: Only allow specific domains
2. **Limit methods**: Only allow needed HTTP methods
3. **Limit headers**: Only allow necessary headers
4. **Credentials**: Use credentials only when needed
5. **Preflight caching**: Cache preflight requests

```java
@Configuration
public class CorsConfig {
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(Arrays.asList("https://app.example.com"));
        config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE"));
        config.setAllowedHeaders(Arrays.asList("Authorization", "Content-Type"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

### Question 5: "How do you implement API throttling per user tier?"

**Answer**:
Tiered rate limiting:

1. **User tiers**: Free, Pro, Enterprise
2. **Different limits**: Higher tiers get more requests
3. **Redis-based**: Store counters in Redis
4. **Headers**: Return remaining requests in headers

```java
public class TieredRateLimitService {
    
    private final Map<UserTier, RateLimit> limits = Map.of(
        UserTier.FREE, new RateLimit(100, Duration.ofMinutes(1)),
        UserTier.PRO, new RateLimit(1000, Duration.ofMinutes(1)),
        UserTier.ENTERPRISE, new RateLimit(10000, Duration.ofMinutes(1))
    );
    
    public boolean isAllowed(String userId, UserTier tier) {
        RateLimit limit = limits.get(tier);
        String key = "ratelimit:" + tier + ":" + userId;
        
        Long count = redisTemplate.opsForValue().increment(key);
        if (count == 1) {
            redisTemplate.expire(key, limit.getWindow().toSeconds(), TimeUnit.SECONDS);
        }
        
        return count <= limit.getMaxRequests();
    }
}
```

---

## 1️⃣1️⃣ One Clean Mental Summary

API security is like a multi-layered checkpoint protecting your application. Every request must pass through: HTTPS encryption (secure transport), authentication (prove identity), authorization (check permissions), input validation (sanitize data), and rate limiting (prevent abuse). Responses are filtered to remove sensitive data and include security headers. Use JWT for stateless APIs, OAuth2 for public APIs, and API keys for simple cases. Never expose secrets in URLs, always validate input, use HTTPS only, and implement proper error handling. Rate limiting prevents abuse, request signing ensures webhook authenticity, and CORS configuration controls cross-origin access. Security is a balance between protection and usability—secure enough to prevent attacks, simple enough for developers to use.

---

