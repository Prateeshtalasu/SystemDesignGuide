# 🛡️ Resilience Patterns

## 0️⃣ Prerequisites

Before diving into resilience patterns, you should understand:

- **Distributed Systems Basics**: Multiple services communicating over a network (covered in Phase 1, Topic 1)
- **Network Failures**: Networks are unreliable. Packets get lost, connections time out, services crash
- **Partial Failures**: In distributed systems, some parts can fail while others continue working. This is fundamentally different from single-machine systems where everything fails together
- **Latency**: The time delay between sending a request and receiving a response. Network calls typically take 1-100ms, but can spike to seconds during problems

**Quick refresher**: When Service A calls Service B over the network, many things can go wrong: Service B might be down, overloaded, slow, or the network between them might be congested. Resilience patterns help Service A survive these failures gracefully.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

In distributed systems, failures are not exceptional. They are the norm. Consider this scenario:

```
Service A → Service B → Service C → Database
```

If Service C becomes slow (not down, just slow), here's what happens without resilience patterns:

1. Service C takes 30 seconds to respond instead of 100ms
2. Service B's threads are blocked waiting for Service C
3. Service B's thread pool exhausts (all threads waiting)
4. Service A's requests to Service B start timing out
5. Service A's threads get blocked
6. The entire system grinds to a halt

**One slow service brought down the entire system.** This is called a **cascading failure**.

### What Systems Looked Like Before

Before resilience patterns became standard practice:

- Services made synchronous calls and waited indefinitely
- One slow dependency could consume all available threads
- Failures propagated upstream, taking down healthy services
- Recovery required manual intervention (restart services, clear queues)
- Systems were fragile. A single component failure meant system-wide outage

### What Breaks Without Resilience Patterns

| Without Pattern | What Happens |
|----------------|--------------|
| No timeouts | Threads wait forever, pool exhaustion |
| No circuit breakers | Repeated calls to failing services waste resources |
| No rate limiting | Traffic spikes overwhelm services |
| No bulkheads | One bad dependency affects all functionality |
| No retries | Transient failures cause permanent errors |
| No fallbacks | Users see errors instead of degraded experience |

### Real Examples of the Problem

**Amazon's 2012 Outage**: A memory leak in one service caused cascading failures across multiple availability zones. Services kept calling the degraded service, exhausting their own resources.

**Netflix's "Dependency of Death"**: Before implementing Hystrix, a single slow third-party service caused Netflix's entire streaming platform to become unresponsive during peak hours.

**Twitter's Fail Whale Era**: Early Twitter had no isolation between features. A spike in one feature (like trending topics) would bring down the entire site.

---

## 2️⃣ Intuition and Mental Model

### The Hospital Emergency Room Analogy

Think of your distributed system as a hospital emergency room:

**Rate Limiting = Triage Nurse**
- Controls how many patients enter per hour
- Turns away non-emergencies during crisis
- Prevents the ER from being overwhelmed

**Circuit Breaker = Quarantine Protocol**
- If a disease outbreak is detected, isolate that ward
- Stop sending new patients to the infected area
- Periodically check if it's safe to reopen

**Bulkhead = Separate Treatment Areas**
- Cardiac patients in one area
- Trauma in another
- Pediatrics separate
- A crisis in trauma doesn't affect cardiac care

**Timeout = Treatment Time Limit**
- If a procedure takes too long, escalate or move on
- Don't let one complex case block all doctors forever

**Retry = Second Opinion**
- If first diagnosis is unclear, try again
- But don't keep trying indefinitely (that's harassment)

**Fallback = Alternative Treatment**
- If the specialist isn't available, the general doctor handles it
- Not ideal, but better than no treatment

This analogy will help us understand each pattern's role in keeping the system "healthy" even when parts are "sick."

---

## 3️⃣ How It Works Internally

### Pattern 1: Rate Limiting

Rate limiting controls how many requests a service accepts in a given time window. It protects services from being overwhelmed.

#### Token Bucket Algorithm

**Core idea**: Imagine a bucket that holds tokens. Each request needs a token. Tokens are added at a fixed rate.

```
Bucket Capacity: 10 tokens
Refill Rate: 2 tokens/second

Time 0s: Bucket has 10 tokens
         5 requests arrive → 5 tokens consumed → 5 tokens remain
         
Time 1s: 2 tokens added → 7 tokens
         3 requests arrive → 3 tokens consumed → 4 tokens remain
         
Time 2s: 2 tokens added → 6 tokens
         10 requests arrive → 6 succeed, 4 rejected (no tokens)
```

**Internal data structure**:
```java
class TokenBucket {
    private final int capacity;           // Maximum tokens
    private final double refillRate;      // Tokens per second
    private double tokens;                // Current tokens
    private long lastRefillTimestamp;     // Last refill time
}
```

**How it works step by step**:

1. Request arrives
2. Calculate time since last refill
3. Add tokens based on elapsed time (capped at capacity)
4. If tokens >= 1, consume one token and allow request
5. If tokens < 1, reject request

**Characteristics**:
- Allows bursts up to bucket capacity
- Smooths out traffic over time
- Memory efficient (just a counter and timestamp)

#### Leaky Bucket Algorithm

**Core idea**: Requests enter a queue (bucket). Requests are processed at a fixed rate. If the queue is full, new requests are dropped.

```
Queue Capacity: 10 requests
Processing Rate: 2 requests/second

Incoming: [R1, R2, R3, R4, R5] (burst of 5)
Queue:    [R1, R2, R3, R4, R5]
          ↓ (leak at 2/sec)
Output:   R1, R2 processed this second
          R3, R4 processed next second
          R5 processed after that
```

**Difference from Token Bucket**:
- Token Bucket: Allows bursts, controls average rate
- Leaky Bucket: Smooths bursts into constant rate output

#### Sliding Window Algorithm

**Core idea**: Track requests in a time window that "slides" forward.

**Fixed Window Problem**:
```
Window 1: 00:00-01:00 → 100 requests (limit 100) ✓
Window 2: 01:00-02:00 → 100 requests (limit 100) ✓

But! 100 requests at 00:59 + 100 requests at 01:01 = 200 requests in 2 seconds
```

**Sliding Window Solution**:
```
Current time: 01:30
Look back 60 seconds to 00:30
Count all requests in [00:30, 01:30]
```

**Implementation approaches**:

1. **Sliding Window Log**: Store timestamp of every request
   - Accurate but memory-intensive
   
2. **Sliding Window Counter**: Weighted average of current and previous window
   - Memory efficient, slight approximation

```java
// Sliding Window Counter
double previousWindowWeight = (60 - secondsIntoCurrentWindow) / 60.0;
double currentWindowWeight = secondsIntoCurrentWindow / 60.0;

double estimatedCount = (previousWindowCount * previousWindowWeight) 
                       + currentWindowCount;
```

---

### Pattern 2: Circuit Breaker

A circuit breaker prevents an application from repeatedly trying to execute an operation that's likely to fail.

#### State Machine

```
         ┌─────────────────────────────────────────────┐
         │                                             │
         ▼                                             │
    ┌─────────┐    failure threshold    ┌──────────┐  │
    │ CLOSED  │ ──────────────────────► │   OPEN   │  │
    │(normal) │                         │(blocking)│  │
    └─────────┘                         └──────────┘  │
         ▲                                    │       │
         │                                    │       │
         │ success                    timeout │       │
         │                                    ▼       │
         │                            ┌───────────┐   │
         └─────────────────────────── │ HALF-OPEN │ ──┘
                                      │ (testing) │ failure
                                      └───────────┘
```

**States explained**:

1. **CLOSED (Normal Operation)**
   - All requests pass through to the service
   - Failures are counted
   - When failure count exceeds threshold, transition to OPEN

2. **OPEN (Blocking)**
   - All requests fail immediately without calling the service
   - A timer starts (reset timeout)
   - After timeout expires, transition to HALF-OPEN

3. **HALF-OPEN (Testing)**
   - Allow a limited number of test requests through
   - If test requests succeed, transition to CLOSED
   - If test requests fail, transition back to OPEN

#### Internal Data Structures

```java
class CircuitBreaker {
    private State state = State.CLOSED;
    private int failureCount = 0;
    private int successCount = 0;
    private long lastFailureTime;
    private long stateChangedTime;
    
    // Configuration
    private final int failureThreshold;        // e.g., 5 failures
    private final long resetTimeoutMs;         // e.g., 30 seconds
    private final int halfOpenMaxRequests;     // e.g., 3 test requests
    private final double failureRateThreshold; // e.g., 50%
}
```

#### How It Works Step by Step

**Scenario: Calling a payment service**

```
Initial State: CLOSED
Failure Threshold: 5 failures in 60 seconds

Request 1: Call payment service → Success → failureCount = 0
Request 2: Call payment service → Success → failureCount = 0
Request 3: Call payment service → Timeout → failureCount = 1
Request 4: Call payment service → Timeout → failureCount = 2
Request 5: Call payment service → Timeout → failureCount = 3
Request 6: Call payment service → Timeout → failureCount = 4
Request 7: Call payment service → Timeout → failureCount = 5

State changes to: OPEN
Reason: Failure threshold (5) reached

Request 8: BLOCKED immediately (no call made)
Request 9: BLOCKED immediately
... (30 seconds pass) ...

State changes to: HALF-OPEN
Reason: Reset timeout expired

Request 10: Call payment service → Success
Request 11: Call payment service → Success
Request 12: Call payment service → Success

State changes to: CLOSED
Reason: Test requests succeeded
```

---

### Pattern 3: Bulkhead

Bulkheads isolate different parts of the system so a failure in one part doesn't cascade to others.

#### Origin of the Name

Ships have bulkheads (watertight compartments). If one compartment floods, the ship doesn't sink because other compartments remain sealed.

#### Types of Bulkheads

**1. Thread Pool Bulkhead**

Each dependency gets its own thread pool:

```
┌─────────────────────────────────────────────────┐
│                  Your Service                    │
│                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌────────┐ │
│  │ Payment Pool │  │ Inventory   │  │ Email  │ │
│  │ (10 threads) │  │ Pool        │  │ Pool   │ │
│  │              │  │ (20 threads)│  │(5 thrd)│ │
│  └──────────────┘  └──────────────┘  └────────┘ │
│         │                 │              │      │
└─────────┼─────────────────┼──────────────┼──────┘
          ▼                 ▼              ▼
    Payment Service   Inventory DB    Email Service
```

If Payment Service is slow and exhausts its 10 threads, Inventory and Email continue working normally.

**2. Semaphore Bulkhead**

Limits concurrent calls without dedicated threads:

```java
Semaphore paymentSemaphore = new Semaphore(10);  // Max 10 concurrent calls

public Response callPayment() {
    if (paymentSemaphore.tryAcquire()) {
        try {
            return paymentClient.call();
        } finally {
            paymentSemaphore.release();
        }
    } else {
        throw new BulkheadFullException("Payment bulkhead full");
    }
}
```

**3. Process/Container Bulkhead**

Different services run in separate containers/processes:

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Container 1 │  │ Container 2 │  │ Container 3 │
│ Order Svc   │  │ Payment Svc │  │ Email Svc   │
│ 2GB RAM     │  │ 1GB RAM     │  │ 512MB RAM   │
└─────────────┘  └─────────────┘  └─────────────┘
```

Memory leak in Payment Service can't consume Order Service's memory.

---

### Pattern 4: Timeout

Timeouts prevent indefinite waiting for responses.

#### Types of Timeouts

**Connection Timeout**: How long to wait to establish a connection
```
Client ──SYN──► Server
       waiting...
       waiting... (connection timeout!)
```

**Read/Socket Timeout**: How long to wait for data after connection is established
```
Client ──Request──► Server
       connected!
       waiting for response...
       waiting... (read timeout!)
```

**Overall/Request Timeout**: Total time for the entire operation
```
Start ──► Connect ──► Send ──► Wait ──► Receive ──► End
|◄─────────────── Overall Timeout ───────────────►|
```

#### Timeout Budgets

In a chain of calls, each service consumes part of the timeout budget:

```
Client Timeout: 3000ms

Client → Service A → Service B → Database
         (500ms)     (1000ms)    (500ms)
         
Service A receives request with 3000ms budget
Service A uses 500ms, passes 2500ms budget to Service B
Service B uses 1000ms, passes 1500ms budget to Database
Database uses 500ms
Total: 2500ms (within budget)
```

---

### Pattern 5: Retry

Retries handle transient failures by attempting the operation again.

#### Exponential Backoff

Instead of retrying immediately, wait progressively longer:

```
Attempt 1: Fail → Wait 100ms
Attempt 2: Fail → Wait 200ms
Attempt 3: Fail → Wait 400ms
Attempt 4: Fail → Wait 800ms
Attempt 5: Fail → Give up

Wait time = baseDelay * (2 ^ attemptNumber)
```

#### Jitter

**Problem with pure exponential backoff**: If 1000 clients fail at the same time, they all retry at the same times, causing "thundering herd."

```
Without Jitter:
Client 1: Retry at 100ms, 200ms, 400ms
Client 2: Retry at 100ms, 200ms, 400ms
Client 3: Retry at 100ms, 200ms, 400ms
→ All hit the server simultaneously!
```

**Solution**: Add randomness (jitter)

```
Full Jitter:
Wait time = random(0, baseDelay * 2^attempt)

Client 1: Retry at 73ms, 189ms, 312ms
Client 2: Retry at 45ms, 156ms, 401ms
Client 3: Retry at 91ms, 203ms, 287ms
→ Spread out!
```

**Equal Jitter** (compromise between predictability and spread):
```
Wait time = (baseDelay * 2^attempt) / 2 + random(0, (baseDelay * 2^attempt) / 2)
```

---

### Pattern 6: Fallback

Fallbacks provide alternative behavior when the primary operation fails.

#### Types of Fallbacks

**1. Static Fallback**: Return a predetermined value
```java
public Product getProduct(String id) {
    try {
        return productService.get(id);
    } catch (Exception e) {
        return Product.DEFAULT;  // Static fallback
    }
}
```

**2. Cache Fallback**: Return stale cached data
```java
public Product getProduct(String id) {
    try {
        Product fresh = productService.get(id);
        cache.put(id, fresh);
        return fresh;
    } catch (Exception e) {
        return cache.get(id);  // Stale but available
    }
}
```

**3. Degraded Fallback**: Return partial data
```java
public ProductPage getProductPage(String id) {
    Product product = getProduct(id);  // Primary data
    
    List<Review> reviews;
    try {
        reviews = reviewService.getReviews(id);
    } catch (Exception e) {
        reviews = Collections.emptyList();  // Page works without reviews
    }
    
    return new ProductPage(product, reviews);
}
```

**4. Alternative Service Fallback**: Use a different service
```java
public Price getPrice(String productId) {
    try {
        return pricingServiceA.getPrice(productId);  // Primary
    } catch (Exception e) {
        return pricingServiceB.getPrice(productId);  // Backup
    }
}
```

---

## 4️⃣ Simulation-First Explanation

Let's trace a single request through a system with all resilience patterns applied.

### Scenario Setup

```
┌──────────┐     ┌──────────────┐     ┌─────────────┐
│  Client  │────►│ Order Service│────►│Payment Svc  │
└──────────┘     └──────────────┘     └─────────────┘
                        │
                        ▼
                 ┌─────────────┐
                 │Inventory Svc│
                 └─────────────┘
```

**Resilience configuration for Order Service**:
- Rate Limit: 100 requests/second
- Circuit Breaker for Payment: 5 failures → open, 30s reset
- Bulkhead: Payment (10 threads), Inventory (20 threads)
- Timeout: Payment (2s), Inventory (1s)
- Retry: 3 attempts with exponential backoff
- Fallback: Return "order pending" if payment fails

### Trace: Successful Request

```
Time 0ms:    Client sends POST /orders
Time 1ms:    Rate limiter checks → 45 requests this second, limit 100 → PASS
Time 2ms:    Request reaches order handler
Time 3ms:    Acquire Payment bulkhead semaphore → 3/10 in use → ACQUIRED
Time 4ms:    Check Payment circuit breaker → CLOSED → PASS
Time 5ms:    Call Payment Service with 2000ms timeout
Time 105ms:  Payment Service responds SUCCESS
Time 106ms:  Release Payment bulkhead semaphore → 2/10 in use
Time 107ms:  Acquire Inventory bulkhead semaphore → 8/20 in use → ACQUIRED
Time 108ms:  Call Inventory Service with 1000ms timeout
Time 158ms:  Inventory Service responds SUCCESS
Time 159ms:  Release Inventory bulkhead semaphore → 7/20 in use
Time 160ms:  Return 201 Created to client

Total time: 160ms
```

### Trace: Payment Service Failing

```
Time 0ms:    Client sends POST /orders
Time 1ms:    Rate limiter → PASS
Time 2ms:    Acquire Payment bulkhead → ACQUIRED (5/10)
Time 3ms:    Check circuit breaker → CLOSED (3 recent failures)
Time 4ms:    Call Payment Service (attempt 1)
Time 2004ms: TIMEOUT after 2000ms
Time 2005ms: Retry decision → attempt 2, wait 100ms
Time 2105ms: Call Payment Service (attempt 2)
Time 4105ms: TIMEOUT after 2000ms
Time 4106ms: Retry decision → attempt 3, wait 200ms
Time 4306ms: Call Payment Service (attempt 3)
Time 6306ms: TIMEOUT after 2000ms
Time 6307ms: All retries exhausted
Time 6308ms: Record failure in circuit breaker → 4 failures now
Time 6309ms: Release Payment bulkhead
Time 6310ms: Execute fallback → return "order pending, payment will be retried"
Time 6311ms: Return 202 Accepted to client

Total time: 6311ms (degraded but not failed)
```

### Trace: Circuit Breaker Opens

```
Previous state: 4 failures recorded, circuit CLOSED

Time 0ms:    Client sends POST /orders
Time 1ms:    Rate limiter → PASS
Time 2ms:    Acquire Payment bulkhead → ACQUIRED
Time 3ms:    Call Payment Service (attempt 1)
Time 2003ms: TIMEOUT
Time 2004ms: Record failure → 5 failures now
Time 2005ms: CIRCUIT BREAKER OPENS (threshold 5 reached)
Time 2006ms: Release bulkhead, execute fallback
Time 2007ms: Return 202 Accepted

--- 5 seconds later ---

Time 5000ms: Another client sends POST /orders
Time 5001ms: Rate limiter → PASS
Time 5002ms: Acquire Payment bulkhead → ACQUIRED
Time 5003ms: Check circuit breaker → OPEN
Time 5004ms: FAIL FAST (no call made)
Time 5005ms: Release bulkhead, execute fallback
Time 5006ms: Return 202 Accepted

Total time: 6ms (fast failure!)
```

### Trace: Circuit Breaker Recovery

```
State: OPEN for 30 seconds, transitioning to HALF-OPEN

Time 0ms:    Client sends POST /orders
Time 1ms:    Rate limiter → PASS
Time 2ms:    Acquire Payment bulkhead → ACQUIRED
Time 3ms:    Check circuit breaker → HALF-OPEN (allowing test request)
Time 4ms:    Call Payment Service (test request)
Time 104ms:  Payment Service responds SUCCESS!
Time 105ms:  Circuit breaker records success (1/3 test requests)
Time 106ms:  Continue with order...

--- Next request ---

Time 200ms:  Another test request → SUCCESS (2/3)

--- Next request ---

Time 300ms:  Another test request → SUCCESS (3/3)
Time 301ms:  CIRCUIT BREAKER CLOSES (all test requests succeeded)

--- Normal operation resumes ---
```

### Trace: Bulkhead Protection

```
State: Payment Service is slow (5 seconds per request)
       9 requests currently waiting in Payment bulkhead

Time 0ms:    Client sends POST /orders
Time 1ms:    Rate limiter → PASS
Time 2ms:    Try to acquire Payment bulkhead → 9/10 in use
Time 3ms:    ACQUIRED (10/10 now)

--- Another request ---

Time 10ms:   New client sends POST /orders
Time 11ms:   Rate limiter → PASS
Time 12ms:   Try to acquire Payment bulkhead → 10/10 in use → REJECTED
Time 13ms:   BulkheadFullException thrown
Time 14ms:   Execute fallback immediately
Time 15ms:   Return 202 Accepted

Note: The 11th request failed fast in 5ms instead of waiting 5+ seconds
      Inventory calls continue working normally (separate bulkhead)
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Netflix: Hystrix (The Pioneer)

Netflix created Hystrix after experiencing cascading failures in their microservices architecture.

**Problem they faced**: Netflix has hundreds of microservices. During peak hours, if one service slowed down, the entire platform became unresponsive.

**Their solution**:
- Every service-to-service call wrapped in Hystrix command
- Thread pool isolation per dependency
- Circuit breakers with automatic recovery
- Real-time dashboards showing circuit states

**Key insight from Netflix**: "The difference between a service that fails gracefully and one that takes down your entire system is whether you've thought about failure modes in advance."

**Note**: Hystrix is now in maintenance mode. Netflix recommends Resilience4j for new projects.

### Amazon: Cell-Based Architecture

Amazon uses bulkheads at the infrastructure level:

```
┌─────────────────────────────────────────────────────┐
│                    AWS Region                        │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │ Cell 1  │  │ Cell 2  │  │ Cell 3  │  │ Cell 4  │ │
│  │ Users   │  │ Users   │  │ Users   │  │ Users   │ │
│  │ A-F     │  │ G-L     │  │ M-R     │  │ S-Z     │ │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘ │
└─────────────────────────────────────────────────────┘
```

Each "cell" is a complete, isolated copy of the service. If Cell 1 has problems, only users A-F are affected. This is bulkhead at massive scale.

### Uber: Rate Limiting at the Edge

Uber implements rate limiting at multiple layers:

1. **Edge (API Gateway)**: Global rate limits per user/API key
2. **Service Level**: Per-service rate limits
3. **Database Level**: Connection pool limits

**Their rate limit hierarchy**:
```
User Rate Limit: 100 requests/minute
├── Ride Service: 50 requests/minute
├── Payment Service: 20 requests/minute
└── Profile Service: 30 requests/minute
```

### Stripe: Idempotency + Retry

Stripe's API is designed for safe retries:

```
POST /v1/charges
Idempotency-Key: abc123

If network fails, client retries with same key
Stripe returns same response (no duplicate charge)
```

Their client libraries implement exponential backoff with jitter automatically.

### Google: Deadline Propagation

Google's gRPC uses deadline propagation across services:

```
Client sets deadline: 5 seconds

Client → Service A (receives 5s deadline)
         Service A uses 1s, forwards 4s deadline to Service B
         Service B uses 0.5s, forwards 3.5s deadline to Service C
         
If deadline expires at any point, entire chain aborts
```

This prevents zombie requests that continue processing after the client has given up.

### Production War Stories

**Slack's Cascading Failure (2022)**: A configuration change caused one service to become slow. Without proper circuit breakers, the slowness cascaded through 50+ services, causing a 4-hour outage. Post-mortem led to mandatory circuit breakers for all service calls.

**GitHub's Database Overload (2020)**: A bug caused retry storms against their database. Thousands of clients retrying simultaneously overwhelmed the database. Solution: Implemented adaptive rate limiting that backs off when error rates increase.

---

## 6️⃣ How to Implement or Apply It

### Dependencies (Maven)

```xml
<!-- Resilience4j - Modern resilience library -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.1.0</version>
</dependency>

<!-- Includes all modules: circuitbreaker, ratelimiter, bulkhead, retry, timelimiter -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-all</artifactId>
    <version>2.1.0</version>
</dependency>

<!-- For metrics and monitoring -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-micrometer</artifactId>
    <version>2.1.0</version>
</dependency>

<!-- AOP support for annotations -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### Configuration (application.yml)

```yaml
resilience4j:
  # Circuit Breaker Configuration
  circuitbreaker:
    instances:
      paymentService:
        registerHealthIndicator: true
        slidingWindowSize: 10                    # Number of calls to consider
        slidingWindowType: COUNT_BASED           # or TIME_BASED
        minimumNumberOfCalls: 5                  # Min calls before calculating failure rate
        failureRateThreshold: 50                 # Percentage to trip circuit
        slowCallRateThreshold: 100               # Percentage of slow calls to trip
        slowCallDurationThreshold: 2s            # What counts as "slow"
        waitDurationInOpenState: 30s             # How long to stay open
        permittedNumberOfCallsInHalfOpenState: 3 # Test calls in half-open
        automaticTransitionFromOpenToHalfOpenEnabled: true
        
      inventoryService:
        slidingWindowSize: 20
        failureRateThreshold: 60
        waitDurationInOpenState: 60s

  # Rate Limiter Configuration
  ratelimiter:
    instances:
      apiRateLimit:
        limitForPeriod: 100              # Number of permits
        limitRefreshPeriod: 1s           # Period to refresh permits
        timeoutDuration: 500ms           # How long to wait for permit
        registerHealthIndicator: true
        
      userRateLimit:
        limitForPeriod: 10
        limitRefreshPeriod: 1s
        timeoutDuration: 0ms             # Fail immediately if no permit

  # Bulkhead Configuration
  bulkhead:
    instances:
      paymentBulkhead:
        maxConcurrentCalls: 10           # Semaphore bulkhead
        maxWaitDuration: 100ms           # How long to wait for permit
        
  # Thread Pool Bulkhead (alternative to semaphore)
  thread-pool-bulkhead:
    instances:
      paymentThreadPool:
        maxThreadPoolSize: 10
        coreThreadPoolSize: 5
        queueCapacity: 20
        keepAliveDuration: 100ms

  # Retry Configuration
  retry:
    instances:
      paymentRetry:
        maxAttempts: 3
        waitDuration: 100ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        enableRandomizedWait: true        # Adds jitter
        randomizedWaitFactor: 0.5
        retryExceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
        ignoreExceptions:
          - com.example.BusinessException

  # Timeout Configuration
  timelimiter:
    instances:
      paymentTimeout:
        timeoutDuration: 2s
        cancelRunningFuture: true
```

### Implementation: Rate Limiter

```java
package com.example.resilience.ratelimit;

import io.github.resilience4j.ratelimiter.RateLimiter;
import io.github.resilience4j.ratelimiter.RateLimiterConfig;
import io.github.resilience4j.ratelimiter.RateLimiterRegistry;

import java.time.Duration;
import java.util.function.Supplier;

/**
 * Rate limiter implementation using Token Bucket algorithm.
 * 
 * How it works:
 * 1. A bucket holds tokens (permits)
 * 2. Each request consumes one token
 * 3. Tokens are refilled at a fixed rate
 * 4. If no tokens available, request waits or is rejected
 */
public class RateLimiterExample {

    private final RateLimiterRegistry registry;
    
    public RateLimiterExample() {
        // Create a registry with default configuration
        RateLimiterConfig defaultConfig = RateLimiterConfig.custom()
            .limitForPeriod(100)                    // 100 requests
            .limitRefreshPeriod(Duration.ofSeconds(1)) // per second
            .timeoutDuration(Duration.ofMillis(500))   // wait up to 500ms for permit
            .build();
            
        this.registry = RateLimiterRegistry.of(defaultConfig);
    }
    
    /**
     * Create a rate limiter for a specific user.
     * This allows per-user rate limiting.
     */
    public RateLimiter createUserRateLimiter(String userId) {
        RateLimiterConfig userConfig = RateLimiterConfig.custom()
            .limitForPeriod(10)                     // 10 requests per user
            .limitRefreshPeriod(Duration.ofSeconds(1))
            .timeoutDuration(Duration.ZERO)         // Fail immediately
            .build();
            
        return registry.rateLimiter("user-" + userId, userConfig);
    }
    
    /**
     * Execute an operation with rate limiting.
     * 
     * @param rateLimiter The rate limiter to use
     * @param operation The operation to execute
     * @return Result of the operation
     * @throws io.github.resilience4j.ratelimiter.RequestNotPermitted if rate limit exceeded
     */
    public <T> T executeWithRateLimit(RateLimiter rateLimiter, Supplier<T> operation) {
        // Decorate the operation with rate limiting
        Supplier<T> decoratedSupplier = RateLimiter.decorateSupplier(rateLimiter, operation);
        
        // Execute - will throw RequestNotPermitted if rate limit exceeded
        return decoratedSupplier.get();
    }
}
```

### Implementation: Circuit Breaker

```java
package com.example.resilience.circuitbreaker;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Duration;
import java.util.function.Supplier;

/**
 * Circuit breaker implementation for protecting against cascading failures.
 * 
 * State transitions:
 * CLOSED → OPEN: When failure rate exceeds threshold
 * OPEN → HALF_OPEN: After wait duration expires
 * HALF_OPEN → CLOSED: When test requests succeed
 * HALF_OPEN → OPEN: When test requests fail
 */
public class CircuitBreakerExample {

    private static final Logger log = LoggerFactory.getLogger(CircuitBreakerExample.class);
    
    private final CircuitBreakerRegistry registry;
    
    public CircuitBreakerExample() {
        // Create registry with custom default configuration
        CircuitBreakerConfig defaultConfig = CircuitBreakerConfig.custom()
            // Sliding window configuration
            .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(10)           // Consider last 10 calls
            .minimumNumberOfCalls(5)         // Need at least 5 calls to calculate rate
            
            // Failure thresholds
            .failureRateThreshold(50)        // Open if 50% fail
            .slowCallRateThreshold(80)       // Open if 80% are slow
            .slowCallDurationThreshold(Duration.ofSeconds(2)) // 2s = slow
            
            // State transition timing
            .waitDurationInOpenState(Duration.ofSeconds(30))  // Stay open 30s
            .permittedNumberOfCallsInHalfOpenState(3)         // 3 test calls
            .automaticTransitionFromOpenToHalfOpenEnabled(true)
            
            // What counts as failure
            .recordExceptions(Exception.class)
            .ignoreExceptions(IllegalArgumentException.class) // Business errors don't count
            
            .build();
            
        this.registry = CircuitBreakerRegistry.of(defaultConfig);
        
        // Register event handlers for monitoring
        registry.getEventPublisher()
            .onEntryAdded(event -> {
                CircuitBreaker cb = event.getAddedEntry();
                cb.getEventPublisher()
                    .onStateTransition(e -> 
                        log.info("Circuit breaker {} transitioned from {} to {}",
                            cb.getName(), e.getStateTransition().getFromState(),
                            e.getStateTransition().getToState()))
                    .onFailureRateExceeded(e ->
                        log.warn("Circuit breaker {} failure rate {} exceeded threshold",
                            cb.getName(), e.getFailureRate()))
                    .onSlowCallRateExceeded(e ->
                        log.warn("Circuit breaker {} slow call rate {} exceeded threshold",
                            cb.getName(), e.getSlowCallRate()));
            });
    }
    
    /**
     * Get or create a circuit breaker for a specific service.
     */
    public CircuitBreaker getCircuitBreaker(String serviceName) {
        return registry.circuitBreaker(serviceName);
    }
    
    /**
     * Execute an operation with circuit breaker protection.
     * 
     * @param circuitBreaker The circuit breaker to use
     * @param operation The operation to execute
     * @param fallback Fallback to use when circuit is open
     * @return Result of operation or fallback
     */
    public <T> T executeWithCircuitBreaker(
            CircuitBreaker circuitBreaker,
            Supplier<T> operation,
            Supplier<T> fallback) {
        
        // Decorate with circuit breaker
        Supplier<T> decoratedSupplier = CircuitBreaker.decorateSupplier(
            circuitBreaker, operation);
        
        try {
            return decoratedSupplier.get();
        } catch (io.github.resilience4j.circuitbreaker.CallNotPermittedException e) {
            // Circuit is open, use fallback
            log.warn("Circuit breaker {} is open, using fallback", circuitBreaker.getName());
            return fallback.get();
        } catch (Exception e) {
            // Operation failed, circuit breaker recorded it
            log.error("Operation failed: {}", e.getMessage());
            return fallback.get();
        }
    }
    
    /**
     * Get current state and metrics of a circuit breaker.
     */
    public void printCircuitBreakerStatus(CircuitBreaker circuitBreaker) {
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        
        log.info("Circuit Breaker: {}", circuitBreaker.getName());
        log.info("  State: {}", circuitBreaker.getState());
        log.info("  Failure rate: {}%", metrics.getFailureRate());
        log.info("  Slow call rate: {}%", metrics.getSlowCallRate());
        log.info("  Buffered calls: {}", metrics.getNumberOfBufferedCalls());
        log.info("  Failed calls: {}", metrics.getNumberOfFailedCalls());
        log.info("  Slow calls: {}", metrics.getNumberOfSlowCalls());
        log.info("  Not permitted calls: {}", metrics.getNumberOfNotPermittedCalls());
    }
}
```

### Implementation: Bulkhead

```java
package com.example.resilience.bulkhead;

import io.github.resilience4j.bulkhead.Bulkhead;
import io.github.resilience4j.bulkhead.BulkheadConfig;
import io.github.resilience4j.bulkhead.BulkheadRegistry;
import io.github.resilience4j.bulkhead.ThreadPoolBulkhead;
import io.github.resilience4j.bulkhead.ThreadPoolBulkheadConfig;
import io.github.resilience4j.bulkhead.ThreadPoolBulkheadRegistry;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Duration;
import java.util.concurrent.CompletableFuture;
import java.util.function.Supplier;

/**
 * Bulkhead implementation for isolating different parts of the system.
 * 
 * Two types:
 * 1. Semaphore Bulkhead: Limits concurrent calls (simpler, less overhead)
 * 2. Thread Pool Bulkhead: Dedicated thread pool per dependency (stronger isolation)
 */
public class BulkheadExample {

    private static final Logger log = LoggerFactory.getLogger(BulkheadExample.class);
    
    private final BulkheadRegistry semaphoreRegistry;
    private final ThreadPoolBulkheadRegistry threadPoolRegistry;
    
    public BulkheadExample() {
        // Semaphore bulkhead configuration
        BulkheadConfig semaphoreConfig = BulkheadConfig.custom()
            .maxConcurrentCalls(10)              // Max 10 concurrent calls
            .maxWaitDuration(Duration.ofMillis(100)) // Wait up to 100ms for permit
            .build();
        this.semaphoreRegistry = BulkheadRegistry.of(semaphoreConfig);
        
        // Thread pool bulkhead configuration
        ThreadPoolBulkheadConfig threadPoolConfig = ThreadPoolBulkheadConfig.custom()
            .maxThreadPoolSize(10)               // Max threads
            .coreThreadPoolSize(5)               // Core threads (always running)
            .queueCapacity(20)                   // Queue size when all threads busy
            .keepAliveDuration(Duration.ofSeconds(60)) // Idle thread timeout
            .build();
        this.threadPoolRegistry = ThreadPoolBulkheadRegistry.of(threadPoolConfig);
    }
    
    /**
     * Execute with semaphore bulkhead.
     * 
     * Use when:
     * - You want to limit concurrent calls without thread overhead
     * - The calling thread should execute the operation
     * - You need simpler resource management
     */
    public <T> T executeWithSemaphoreBulkhead(String name, Supplier<T> operation) {
        Bulkhead bulkhead = semaphoreRegistry.bulkhead(name);
        
        // Log current state
        log.debug("Bulkhead {}: {} available of {} max concurrent calls",
            name,
            bulkhead.getMetrics().getAvailableConcurrentCalls(),
            bulkhead.getMetrics().getMaxAllowedConcurrentCalls());
        
        // Decorate and execute
        Supplier<T> decoratedSupplier = Bulkhead.decorateSupplier(bulkhead, operation);
        
        try {
            return decoratedSupplier.get();
        } catch (io.github.resilience4j.bulkhead.BulkheadFullException e) {
            log.warn("Bulkhead {} is full, rejecting request", name);
            throw e;
        }
    }
    
    /**
     * Execute with thread pool bulkhead.
     * 
     * Use when:
     * - You need stronger isolation (dedicated threads)
     * - Operations are CPU-bound or blocking
     * - You want to prevent one dependency from consuming all threads
     * 
     * Returns CompletableFuture because execution is asynchronous.
     */
    public <T> CompletableFuture<T> executeWithThreadPoolBulkhead(
            String name, Supplier<T> operation) {
        
        ThreadPoolBulkhead bulkhead = threadPoolRegistry.bulkhead(name);
        
        // Log current state
        ThreadPoolBulkhead.Metrics metrics = bulkhead.getMetrics();
        log.debug("Thread pool bulkhead {}: {} threads active, {} in queue, {} available",
            name,
            metrics.getActiveThreadCount(),
            metrics.getQueueDepth(),
            metrics.getRemainingQueueCapacity());
        
        // Execute asynchronously in the bulkhead's thread pool
        return bulkhead.executeSupplier(operation);
    }
    
    /**
     * Example: Isolating payment and inventory calls.
     */
    public void demonstrateIsolation() {
        // Payment service gets its own bulkhead
        Bulkhead paymentBulkhead = semaphoreRegistry.bulkhead("payment", 
            BulkheadConfig.custom()
                .maxConcurrentCalls(10)
                .build());
        
        // Inventory service gets its own bulkhead
        Bulkhead inventoryBulkhead = semaphoreRegistry.bulkhead("inventory",
            BulkheadConfig.custom()
                .maxConcurrentCalls(20)
                .build());
        
        // Even if payment is slow and uses all 10 permits,
        // inventory still has its 20 permits available
    }
}
```

### Implementation: Retry with Exponential Backoff

```java
package com.example.resilience.retry;

import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.RetryConfig;
import io.github.resilience4j.retry.RetryRegistry;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.net.SocketTimeoutException;
import java.time.Duration;
import java.util.function.Supplier;

/**
 * Retry implementation with exponential backoff and jitter.
 * 
 * Key concepts:
 * - Exponential backoff: Wait time doubles with each retry
 * - Jitter: Random variation to prevent thundering herd
 * - Selective retry: Only retry on transient failures
 */
public class RetryExample {

    private static final Logger log = LoggerFactory.getLogger(RetryExample.class);
    
    private final RetryRegistry registry;
    
    public RetryExample() {
        // Create retry configuration
        RetryConfig defaultConfig = RetryConfig.custom()
            .maxAttempts(3)                      // Total attempts (1 initial + 2 retries)
            .waitDuration(Duration.ofMillis(100)) // Base wait time
            
            // Exponential backoff
            .intervalFunction(io.github.resilience4j.core.IntervalFunction
                .ofExponentialBackoff(
                    Duration.ofMillis(100),      // Initial interval
                    2.0))                        // Multiplier
            
            // Or with jitter (recommended for production)
            // .intervalFunction(io.github.resilience4j.core.IntervalFunction
            //     .ofExponentialRandomBackoff(
            //         Duration.ofMillis(100),   // Initial interval
            //         2.0,                      // Multiplier
            //         0.5))                     // Randomization factor
            
            // Only retry on these exceptions
            .retryExceptions(
                IOException.class,
                SocketTimeoutException.class)
            
            // Never retry on these (business logic errors)
            .ignoreExceptions(
                IllegalArgumentException.class,
                IllegalStateException.class)
            
            // Custom retry predicate (optional)
            .retryOnResult(response -> response == null)
            
            .build();
            
        this.registry = RetryRegistry.of(defaultConfig);
        
        // Register event handlers
        registry.getEventPublisher()
            .onEntryAdded(event -> {
                Retry retry = event.getAddedEntry();
                retry.getEventPublisher()
                    .onRetry(e -> log.info("Retry attempt {} for {}",
                        e.getNumberOfRetryAttempts(), retry.getName()))
                    .onSuccess(e -> log.info("Retry {} succeeded after {} attempts",
                        retry.getName(), e.getNumberOfRetryAttempts()))
                    .onError(e -> log.error("Retry {} failed after {} attempts",
                        retry.getName(), e.getNumberOfRetryAttempts()));
            });
    }
    
    /**
     * Execute an operation with retry.
     */
    public <T> T executeWithRetry(String name, Supplier<T> operation) {
        Retry retry = registry.retry(name);
        
        Supplier<T> decoratedSupplier = Retry.decorateSupplier(retry, operation);
        
        return decoratedSupplier.get();
    }
    
    /**
     * Custom retry configuration for specific operations.
     */
    public <T> T executeWithCustomRetry(
            Supplier<T> operation,
            int maxAttempts,
            Duration initialDelay,
            Class<? extends Throwable>... retryOn) {
        
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(maxAttempts)
            .intervalFunction(io.github.resilience4j.core.IntervalFunction
                .ofExponentialRandomBackoff(initialDelay, 2.0, 0.5))
            .retryExceptions(retryOn)
            .build();
        
        Retry retry = Retry.of("custom", config);
        
        return Retry.decorateSupplier(retry, operation).get();
    }
}
```

### Implementation: Combined Patterns (Production Service)

```java
package com.example.resilience.service;

import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.ratelimiter.annotation.RateLimiter;
import io.github.resilience4j.retry.annotation.Retry;
import io.github.resilience4j.timelimiter.annotation.TimeLimiter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.concurrent.CompletableFuture;

/**
 * Production service demonstrating all resilience patterns combined.
 * 
 * Order of decorators (outer to inner):
 * 1. Rate Limiter - First line of defense, limits incoming traffic
 * 2. Bulkhead - Limits concurrent executions
 * 3. Time Limiter - Limits execution time
 * 4. Circuit Breaker - Fails fast when service is down
 * 5. Retry - Retries on transient failures
 * 
 * Note: Annotations are processed in reverse order of declaration.
 */
@Service
public class PaymentService {

    private static final Logger log = LoggerFactory.getLogger(PaymentService.class);
    
    private final RestTemplate restTemplate;
    
    public PaymentService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }
    
    /**
     * Process payment with full resilience stack.
     * 
     * Annotation order matters! They are applied in reverse order:
     * - @Retry is innermost (applied first)
     * - @CircuitBreaker wraps retry
     * - @Bulkhead wraps circuit breaker
     * - @RateLimiter is outermost (applied last)
     */
    @RateLimiter(name = "paymentRateLimiter", fallbackMethod = "paymentRateLimitFallback")
    @Bulkhead(name = "paymentBulkhead", fallbackMethod = "paymentBulkheadFallback")
    @CircuitBreaker(name = "paymentCircuitBreaker", fallbackMethod = "paymentCircuitBreakerFallback")
    @Retry(name = "paymentRetry")
    public PaymentResponse processPayment(PaymentRequest request) {
        log.info("Processing payment for order: {}", request.getOrderId());
        
        // Actual HTTP call to payment provider
        return restTemplate.postForObject(
            "https://payment-provider.com/api/charge",
            request,
            PaymentResponse.class);
    }
    
    /**
     * Async version with time limiter.
     * TimeLimiter requires CompletableFuture return type.
     */
    @TimeLimiter(name = "paymentTimeout", fallbackMethod = "paymentTimeoutFallback")
    @CircuitBreaker(name = "paymentCircuitBreaker", fallbackMethod = "paymentCircuitBreakerFallbackAsync")
    @Retry(name = "paymentRetry")
    public CompletableFuture<PaymentResponse> processPaymentAsync(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            log.info("Processing async payment for order: {}", request.getOrderId());
            return restTemplate.postForObject(
                "https://payment-provider.com/api/charge",
                request,
                PaymentResponse.class);
        });
    }
    
    // ==================== Fallback Methods ====================
    
    /**
     * Fallback when rate limit exceeded.
     * Must have same signature + Exception parameter.
     */
    public PaymentResponse paymentRateLimitFallback(PaymentRequest request, Exception e) {
        log.warn("Rate limit exceeded for order: {}", request.getOrderId());
        return PaymentResponse.builder()
            .orderId(request.getOrderId())
            .status(PaymentStatus.RATE_LIMITED)
            .message("Too many requests. Please try again later.")
            .build();
    }
    
    /**
     * Fallback when bulkhead is full.
     */
    public PaymentResponse paymentBulkheadFallback(PaymentRequest request, Exception e) {
        log.warn("Bulkhead full for order: {}", request.getOrderId());
        return PaymentResponse.builder()
            .orderId(request.getOrderId())
            .status(PaymentStatus.PENDING)
            .message("System busy. Payment queued for processing.")
            .build();
    }
    
    /**
     * Fallback when circuit breaker is open.
     */
    public PaymentResponse paymentCircuitBreakerFallback(PaymentRequest request, Exception e) {
        log.warn("Circuit breaker open for order: {}", request.getOrderId());
        return PaymentResponse.builder()
            .orderId(request.getOrderId())
            .status(PaymentStatus.PENDING)
            .message("Payment service temporarily unavailable. Will retry automatically.")
            .build();
    }
    
    /**
     * Async fallback for circuit breaker.
     */
    public CompletableFuture<PaymentResponse> paymentCircuitBreakerFallbackAsync(
            PaymentRequest request, Exception e) {
        return CompletableFuture.completedFuture(
            paymentCircuitBreakerFallback(request, e));
    }
    
    /**
     * Fallback when timeout exceeded.
     */
    public CompletableFuture<PaymentResponse> paymentTimeoutFallback(
            PaymentRequest request, Exception e) {
        log.warn("Timeout exceeded for order: {}", request.getOrderId());
        return CompletableFuture.completedFuture(
            PaymentResponse.builder()
                .orderId(request.getOrderId())
                .status(PaymentStatus.PENDING)
                .message("Payment processing taking longer than expected. Will notify when complete.")
                .build());
    }
}

// Supporting classes
record PaymentRequest(String orderId, double amount, String currency) {
    public String getOrderId() { return orderId; }
}

record PaymentResponse(String orderId, PaymentStatus status, String message) {
    public static PaymentResponseBuilder builder() {
        return new PaymentResponseBuilder();
    }
    
    static class PaymentResponseBuilder {
        private String orderId;
        private PaymentStatus status;
        private String message;
        
        public PaymentResponseBuilder orderId(String orderId) {
            this.orderId = orderId;
            return this;
        }
        
        public PaymentResponseBuilder status(PaymentStatus status) {
            this.status = status;
            return this;
        }
        
        public PaymentResponseBuilder message(String message) {
            this.message = message;
            return this;
        }
        
        public PaymentResponse build() {
            return new PaymentResponse(orderId, status, message);
        }
    }
}

enum PaymentStatus {
    SUCCESS, FAILED, PENDING, RATE_LIMITED
}
```

### Implementation: Manual Token Bucket Rate Limiter (From Scratch)

```java
package com.example.resilience.ratelimit;

import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Manual implementation of Token Bucket rate limiter.
 * 
 * This shows how the algorithm works internally.
 * In production, use Resilience4j or similar library.
 */
public class TokenBucketRateLimiter {
    
    private final long capacity;           // Maximum tokens
    private final double refillRate;       // Tokens per nanosecond
    private final ReentrantLock lock;
    
    private double tokens;                 // Current tokens (can be fractional)
    private long lastRefillTimestamp;      // Last refill time in nanoseconds
    
    /**
     * Create a token bucket rate limiter.
     * 
     * @param capacity Maximum tokens (burst capacity)
     * @param refillRatePerSecond Tokens added per second
     */
    public TokenBucketRateLimiter(long capacity, double refillRatePerSecond) {
        this.capacity = capacity;
        this.refillRate = refillRatePerSecond / 1_000_000_000.0; // Convert to per-nanosecond
        this.tokens = capacity;  // Start full
        this.lastRefillTimestamp = System.nanoTime();
        this.lock = new ReentrantLock();
    }
    
    /**
     * Try to acquire a permit.
     * 
     * @return true if permit acquired, false if rate limited
     */
    public boolean tryAcquire() {
        return tryAcquire(1);
    }
    
    /**
     * Try to acquire multiple permits.
     * 
     * @param permits Number of permits to acquire
     * @return true if permits acquired, false if rate limited
     */
    public boolean tryAcquire(int permits) {
        lock.lock();
        try {
            refill();
            
            if (tokens >= permits) {
                tokens -= permits;
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }
    
    /**
     * Refill tokens based on elapsed time.
     */
    private void refill() {
        long now = System.nanoTime();
        long elapsed = now - lastRefillTimestamp;
        
        // Calculate tokens to add
        double tokensToAdd = elapsed * refillRate;
        
        // Add tokens, capped at capacity
        tokens = Math.min(capacity, tokens + tokensToAdd);
        
        lastRefillTimestamp = now;
    }
    
    /**
     * Get current token count (for monitoring).
     */
    public double getAvailableTokens() {
        lock.lock();
        try {
            refill();
            return tokens;
        } finally {
            lock.unlock();
        }
    }
    
    // Usage example
    public static void main(String[] args) throws InterruptedException {
        // 10 requests per second, burst of 5
        TokenBucketRateLimiter limiter = new TokenBucketRateLimiter(5, 10);
        
        // Burst: first 5 should succeed
        for (int i = 0; i < 7; i++) {
            boolean allowed = limiter.tryAcquire();
            System.out.printf("Request %d: %s (tokens: %.2f)%n",
                i + 1, allowed ? "ALLOWED" : "DENIED", limiter.getAvailableTokens());
        }
        
        // Wait for refill
        System.out.println("\nWaiting 500ms for refill...\n");
        Thread.sleep(500);
        
        // Should have ~5 tokens again
        for (int i = 0; i < 3; i++) {
            boolean allowed = limiter.tryAcquire();
            System.out.printf("Request %d: %s (tokens: %.2f)%n",
                i + 8, allowed ? "ALLOWED" : "DENIED", limiter.getAvailableTokens());
        }
    }
}
```

### Implementation: Manual Circuit Breaker (From Scratch)

```java
package com.example.resilience.circuitbreaker;

import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReference;
import java.util.function.Supplier;

/**
 * Manual implementation of Circuit Breaker pattern.
 * 
 * This shows how the state machine works internally.
 * In production, use Resilience4j or similar library.
 */
public class SimpleCircuitBreaker {
    
    public enum State {
        CLOSED,     // Normal operation, counting failures
        OPEN,       // Blocking all calls
        HALF_OPEN   // Testing if service recovered
    }
    
    private final String name;
    private final int failureThreshold;
    private final Duration resetTimeout;
    private final int halfOpenMaxCalls;
    
    private final AtomicReference<State> state = new AtomicReference<>(State.CLOSED);
    private final AtomicInteger failureCount = new AtomicInteger(0);
    private final AtomicInteger halfOpenCallCount = new AtomicInteger(0);
    private volatile Instant lastFailureTime;
    private volatile Instant openedAt;
    
    public SimpleCircuitBreaker(String name, int failureThreshold, 
                                Duration resetTimeout, int halfOpenMaxCalls) {
        this.name = name;
        this.failureThreshold = failureThreshold;
        this.resetTimeout = resetTimeout;
        this.halfOpenMaxCalls = halfOpenMaxCalls;
    }
    
    /**
     * Execute an operation with circuit breaker protection.
     */
    public <T> T execute(Supplier<T> operation, Supplier<T> fallback) {
        // Check if we should transition from OPEN to HALF_OPEN
        if (state.get() == State.OPEN && shouldAttemptReset()) {
            state.compareAndSet(State.OPEN, State.HALF_OPEN);
            halfOpenCallCount.set(0);
        }
        
        switch (state.get()) {
            case CLOSED:
                return executeInClosedState(operation, fallback);
                
            case OPEN:
                // Fast fail
                System.out.printf("[%s] Circuit OPEN - fast failing%n", name);
                return fallback.get();
                
            case HALF_OPEN:
                return executeInHalfOpenState(operation, fallback);
                
            default:
                throw new IllegalStateException("Unknown state: " + state.get());
        }
    }
    
    private <T> T executeInClosedState(Supplier<T> operation, Supplier<T> fallback) {
        try {
            T result = operation.get();
            // Success - reset failure count
            failureCount.set(0);
            return result;
        } catch (Exception e) {
            // Failure - increment count
            int failures = failureCount.incrementAndGet();
            lastFailureTime = Instant.now();
            
            System.out.printf("[%s] Failure %d/%d in CLOSED state%n", 
                name, failures, failureThreshold);
            
            if (failures >= failureThreshold) {
                // Trip the circuit
                state.set(State.OPEN);
                openedAt = Instant.now();
                System.out.printf("[%s] Circuit OPENED%n", name);
            }
            
            return fallback.get();
        }
    }
    
    private <T> T executeInHalfOpenState(Supplier<T> operation, Supplier<T> fallback) {
        int callNumber = halfOpenCallCount.incrementAndGet();
        
        if (callNumber > halfOpenMaxCalls) {
            // Already have enough test calls in progress
            System.out.printf("[%s] HALF_OPEN - max test calls reached, fast failing%n", name);
            return fallback.get();
        }
        
        try {
            T result = operation.get();
            
            System.out.printf("[%s] Test call %d/%d succeeded%n", 
                name, callNumber, halfOpenMaxCalls);
            
            if (callNumber >= halfOpenMaxCalls) {
                // All test calls succeeded, close the circuit
                state.set(State.CLOSED);
                failureCount.set(0);
                System.out.printf("[%s] Circuit CLOSED%n", name);
            }
            
            return result;
        } catch (Exception e) {
            // Test call failed, reopen circuit
            state.set(State.OPEN);
            openedAt = Instant.now();
            System.out.printf("[%s] Test call failed, circuit REOPENED%n", name);
            
            return fallback.get();
        }
    }
    
    private boolean shouldAttemptReset() {
        if (openedAt == null) return false;
        return Duration.between(openedAt, Instant.now()).compareTo(resetTimeout) >= 0;
    }
    
    public State getState() {
        return state.get();
    }
    
    // Usage example
    public static void main(String[] args) throws InterruptedException {
        SimpleCircuitBreaker cb = new SimpleCircuitBreaker(
            "payment", 3, Duration.ofSeconds(5), 2);
        
        // Simulate failing service
        Supplier<String> failingOperation = () -> {
            throw new RuntimeException("Service unavailable");
        };
        
        Supplier<String> fallback = () -> "FALLBACK";
        
        // Trip the circuit
        for (int i = 0; i < 5; i++) {
            String result = cb.execute(failingOperation, fallback);
            System.out.printf("Result: %s, State: %s%n%n", result, cb.getState());
        }
        
        // Wait for reset timeout
        System.out.println("Waiting 6 seconds for reset timeout...\n");
        Thread.sleep(6000);
        
        // Simulate recovered service
        Supplier<String> workingOperation = () -> "SUCCESS";
        
        // Test calls in HALF_OPEN
        for (int i = 0; i < 3; i++) {
            String result = cb.execute(workingOperation, fallback);
            System.out.printf("Result: %s, State: %s%n%n", result, cb.getState());
        }
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Tradeoffs

| Pattern | Benefit | Cost |
|---------|---------|------|
| Rate Limiting | Protects from overload | Rejects legitimate traffic during spikes |
| Circuit Breaker | Fast failure, prevents cascading | May reject requests when service is actually up |
| Bulkhead | Isolation between dependencies | Resource overhead (threads, memory) |
| Timeout | Prevents indefinite waiting | May abort operations that would have succeeded |
| Retry | Handles transient failures | Can amplify load on struggling services |
| Fallback | Graceful degradation | Returns potentially stale/incomplete data |

### Common Mistakes

**1. Retry Without Backoff**
```java
// BAD: Hammers the service
for (int i = 0; i < 3; i++) {
    try {
        return service.call();
    } catch (Exception e) {
        // Retry immediately
    }
}

// GOOD: Exponential backoff with jitter
```

**2. Retry on Non-Idempotent Operations**
```java
// DANGEROUS: May create duplicate payments
@Retry(name = "payment")
public void createPayment(PaymentRequest request) {
    paymentApi.charge(request);  // Not idempotent!
}

// SAFE: Use idempotency key
@Retry(name = "payment")
public void createPayment(PaymentRequest request) {
    paymentApi.charge(request.getIdempotencyKey(), request);
}
```

**3. Circuit Breaker Threshold Too Sensitive**
```java
// BAD: Opens on first failure
CircuitBreakerConfig.custom()
    .slidingWindowSize(1)
    .failureRateThreshold(100)  // 1 failure = open
    .build();

// GOOD: Requires pattern of failures
CircuitBreakerConfig.custom()
    .slidingWindowSize(10)
    .minimumNumberOfCalls(5)
    .failureRateThreshold(50)
    .build();
```

**4. Timeout Longer Than Client Timeout**
```java
// BAD: Service timeout > client timeout
// Client gives up at 5s, but service keeps processing
serviceTimeout = 10s;
clientTimeout = 5s;

// Result: Wasted resources processing requests nobody is waiting for

// GOOD: Service timeout < client timeout
serviceTimeout = 4s;
clientTimeout = 5s;
```

**5. No Fallback for Circuit Breaker**
```java
// BAD: Exception propagates to user
@CircuitBreaker(name = "payment")
public PaymentResponse processPayment(PaymentRequest request) {
    return paymentApi.charge(request);
}
// When circuit opens, CallNotPermittedException thrown

// GOOD: Graceful fallback
@CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
public PaymentResponse processPayment(PaymentRequest request) {
    return paymentApi.charge(request);
}
```

**6. Bulkhead Too Small**
```java
// BAD: Normal traffic exceeds bulkhead
// Average concurrent calls: 50
// Bulkhead limit: 10
// Result: 80% of requests rejected

// GOOD: Size based on actual traffic patterns
// Peak concurrent calls: 100
// Bulkhead limit: 150 (50% headroom)
```

### Performance Gotchas

**Thread Pool Bulkhead Overhead**
- Each call requires thread handoff
- Context switching cost
- Use semaphore bulkhead for simple limiting

**Retry Storm**
- If many clients retry simultaneously, load multiplies
- 1000 clients × 3 retries = 3000 requests
- Always use jitter

**Circuit Breaker State Storage**
- In-memory state lost on restart
- Consider distributed state for critical circuits
- Or accept that circuit resets on restart

### Security Considerations

**Rate Limiting Bypass**
- Rate limit by user ID, not just IP (proxies share IPs)
- Include API key in rate limit key
- Consider distributed rate limiting for multi-instance deployments

**Fallback Data Exposure**
- Cached fallback data might be sensitive
- Ensure fallback doesn't bypass authorization
- Log fallback usage for audit

---

## 8️⃣ When NOT to Use This

### Rate Limiting

**Don't use when**:
- Internal service-to-service calls within trusted network
- Batch processing systems where throughput matters more than fairness
- Event-driven systems (use backpressure instead)

**Better alternatives**:
- Backpressure mechanisms (reactive streams)
- Queue-based load leveling
- Auto-scaling

### Circuit Breaker

**Don't use when**:
- Calling local methods or in-process components
- Operations that must always be attempted (critical path)
- Idempotent operations where retry is cheap

**Better alternatives**:
- Simple retry for transient failures
- Timeout alone for slow operations
- Bulkhead for isolation without blocking

### Bulkhead

**Don't use when**:
- Single dependency system
- Already using container-level isolation (Kubernetes resource limits)
- Thread overhead is unacceptable

**Better alternatives**:
- Container resource limits
- Kubernetes pod quotas
- Simple connection pool limits

### Retry

**Don't use when**:
- Operation is not idempotent
- Failure is deterministic (bad input, authorization failure)
- Real-time systems where latency matters more than success

**Better alternatives**:
- Async retry with queue
- Dead letter queue for manual retry
- Compensation transaction

---

## 9️⃣ Comparison with Alternatives

### Rate Limiting Approaches

| Approach | Pros | Cons | Use When |
|----------|------|------|----------|
| Token Bucket | Allows bursts, simple | Burst can overwhelm | API rate limiting |
| Leaky Bucket | Smooth output rate | No burst allowed | Streaming, real-time |
| Sliding Window | Accurate, no boundary issues | More memory | Strict rate enforcement |
| Fixed Window | Simple, low memory | Boundary burst problem | Approximate limiting |

### Circuit Breaker vs Retry

| Aspect | Circuit Breaker | Retry |
|--------|----------------|-------|
| Purpose | Prevent wasted calls | Handle transient failures |
| Behavior | Fails fast when open | Keeps trying |
| Best for | Prolonged outages | Brief glitches |
| Overhead | State tracking | Multiple calls |
| Combine? | Yes! Retry inside circuit breaker |

### Bulkhead Implementations

| Type | Isolation | Overhead | Use When |
|------|-----------|----------|----------|
| Semaphore | Limits concurrency | Low | Most cases |
| Thread Pool | Dedicated threads | High | CPU-bound work |
| Process | Complete isolation | Highest | Critical isolation |

### Resilience4j vs Hystrix

| Aspect | Resilience4j | Hystrix |
|--------|--------------|---------|
| Status | Active | Maintenance mode |
| Dependencies | Lightweight | Heavy (RxJava) |
| Configuration | Flexible | Opinionated |
| Java Version | 8+ (17+ recommended) | 8 |
| Spring Boot | Native support | Via wrapper |
| Recommendation | Use for new projects | Migrate away |

---

## 🔟 Interview Follow-up Questions WITH Answers

### L4 (Entry Level) Questions

**Q1: What is a circuit breaker and why do we need it?**

**A**: A circuit breaker is a design pattern that prevents an application from repeatedly trying to execute an operation that's likely to fail. It works like an electrical circuit breaker:

1. **CLOSED state**: Normal operation. Requests flow through. Failures are counted.
2. **OPEN state**: When failures exceed a threshold, the circuit "trips." All requests fail immediately without calling the downstream service.
3. **HALF-OPEN state**: After a timeout, the circuit allows a few test requests. If they succeed, circuit closes. If they fail, circuit reopens.

We need it because:
- Prevents cascading failures (one slow service doesn't bring down everything)
- Saves resources (no threads waiting for timeouts)
- Gives failing services time to recover
- Provides fast failure instead of slow failure

---

**Q2: Explain exponential backoff with jitter.**

**A**: Exponential backoff is a retry strategy where the wait time between retries increases exponentially:

```
Attempt 1: Wait 100ms
Attempt 2: Wait 200ms (100 × 2)
Attempt 3: Wait 400ms (200 × 2)
Attempt 4: Wait 800ms (400 × 2)
```

The problem is "thundering herd": if 1000 clients fail at the same time, they all retry at the same times (100ms, 200ms, 400ms), causing synchronized load spikes.

Jitter adds randomness to spread out retries:

```
Full jitter: Wait = random(0, calculated_delay)
Equal jitter: Wait = (calculated_delay / 2) + random(0, calculated_delay / 2)
```

Example with full jitter:
```
Client 1: Wait 73ms, 189ms, 312ms
Client 2: Wait 45ms, 156ms, 401ms
Client 3: Wait 91ms, 203ms, 287ms
```

Now retries are spread out, reducing the load spike on the recovering service.

---

### L5 (Mid Level) Questions

**Q3: How would you implement rate limiting for a distributed system with multiple instances?**

**A**: For distributed rate limiting, we need shared state across instances. Here are the approaches:

**1. Centralized (Redis-based)**
```
Instance A ──┐
Instance B ──┼──► Redis ──► Rate limit state
Instance C ──┘

Pros: Accurate, consistent
Cons: Redis becomes SPOF, added latency
```

Implementation with Redis:
```java
// Using Redis INCR with TTL
public boolean isAllowed(String key, int limit, int windowSeconds) {
    String redisKey = "ratelimit:" + key;
    Long count = redis.incr(redisKey);
    if (count == 1) {
        redis.expire(redisKey, windowSeconds);
    }
    return count <= limit;
}
```

**2. Sliding Window with Redis**
```java
// More accurate, uses sorted set
public boolean isAllowed(String key, int limit, int windowMs) {
    long now = System.currentTimeMillis();
    long windowStart = now - windowMs;
    
    // Remove old entries, add new entry, count
    redis.zremrangeByScore(key, 0, windowStart);
    redis.zadd(key, now, UUID.randomUUID().toString());
    long count = redis.zcard(key);
    redis.expire(key, windowMs / 1000 + 1);
    
    return count <= limit;
}
```

**3. Local + Coordination**
- Each instance gets a portion of the limit
- Periodically sync with central coordinator
- Pros: Lower latency, works if coordinator is down
- Cons: Less accurate

**4. Token Bucket with Redis**
- Store bucket state in Redis
- Use Lua script for atomic refill + acquire

The choice depends on accuracy requirements, latency tolerance, and failure handling needs.

---

**Q4: Design a bulkhead strategy for a service that calls 5 different downstream services.**

**A**: I would implement a layered bulkhead strategy:

**1. Analyze dependencies**
```
Service A: Critical (payment) - 100ms avg, 2s max
Service B: Critical (inventory) - 50ms avg, 1s max
Service C: Non-critical (recommendations) - 200ms avg, 5s max
Service D: Non-critical (analytics) - 500ms avg, 10s max
Service E: Critical (auth) - 20ms avg, 500ms max
```

**2. Size bulkheads based on traffic**
```
Expected concurrent requests: 1000
Thread pool size: 200

Allocation:
- Payment: 50 threads (critical, slow)
- Inventory: 40 threads (critical, fast)
- Auth: 30 threads (critical, very fast)
- Recommendations: 20 threads (non-critical)
- Analytics: 10 threads (non-critical, can fail)
- Reserve: 50 threads (for spikes)
```

**3. Choose bulkhead type**
```java
// Semaphore for fast services (Auth)
@Bulkhead(name = "auth", type = Bulkhead.Type.SEMAPHORE)

// Thread pool for slow/blocking services (Payment)
@Bulkhead(name = "payment", type = Bulkhead.Type.THREADPOOL)
```

**4. Configure fallbacks**
```java
// Critical services: Queue for retry
@Bulkhead(name = "payment", fallbackMethod = "queuePayment")

// Non-critical: Return empty/default
@Bulkhead(name = "recommendations", fallbackMethod = "emptyRecommendations")
```

**5. Monitoring**
- Alert when bulkhead utilization > 80%
- Track rejection rate per bulkhead
- Auto-scale if sustained high utilization

---

### L6 (Senior Level) Questions

**Q5: How would you handle a scenario where circuit breakers across multiple services open simultaneously during a partial outage?**

**A**: This is a complex scenario that requires coordinated resilience. Here's my approach:

**1. Understand the cascade**
```
Database partial outage
    ↓
Service A circuit opens (DB calls failing)
    ↓
Service B circuit opens (calls to A failing)
    ↓
Service C circuit opens (calls to B failing)
    ↓
Complete system unavailable
```

**2. Implement circuit breaker coordination**

**Approach A: Hierarchical Circuit Breakers**
```java
// Parent circuit breaker for infrastructure
@CircuitBreaker(name = "infrastructure")
public class InfrastructureHealth {
    // Monitors DB, cache, message queue health
}

// Child circuit breakers inherit parent state
@CircuitBreaker(name = "serviceA", parent = "infrastructure")
public class ServiceA { }
```

When infrastructure circuit opens, all child circuits fail fast without individual failures.

**Approach B: Health Aggregation**
```java
// Central health aggregator
public class SystemHealth {
    public HealthStatus getOverallHealth() {
        // Aggregate health from all services
        // If critical mass of circuits open, declare system degraded
    }
}

// Services check system health before attempting calls
if (systemHealth.isDegraded()) {
    return fallback();
}
```

**3. Implement graceful degradation tiers**

```
Tier 1 (Normal): All features available
Tier 2 (Degraded): Non-critical features disabled
Tier 3 (Minimal): Only read operations
Tier 4 (Maintenance): Static error page
```

```java
public Response handleRequest(Request req) {
    switch (systemHealth.getTier()) {
        case NORMAL:
            return fullProcessing(req);
        case DEGRADED:
            return processWithoutRecommendations(req);
        case MINIMAL:
            return readOnlyResponse(req);
        case MAINTENANCE:
            return maintenancePage();
    }
}
```

**4. Implement coordinated recovery**

```java
// Staggered circuit breaker reset
// Don't let all circuits try to recover at once
public Duration getResetTimeout(String serviceName) {
    int hash = serviceName.hashCode() % 10;
    return Duration.ofSeconds(30 + hash * 5); // 30-75 seconds
}
```

**5. Add circuit breaker dashboard**
- Real-time view of all circuit states
- Ability to manually open/close circuits
- Historical view of circuit transitions
- Correlation with infrastructure events

---

**Q6: Design a rate limiting system that handles 1 million requests per second across 100 servers with sub-millisecond latency.**

**A**: At this scale, centralized rate limiting won't work. Here's my design:

**1. Architecture Overview**
```
┌─────────────────────────────────────────────────────────┐
│                    Load Balancer                         │
└─────────────────────────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   ┌─────────┐        ┌─────────┐        ┌─────────┐
   │Server 1 │        │Server 2 │        │Server N │
   │Local RL │        │Local RL │        │Local RL │
   └─────────┘        └─────────┘        └─────────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           ▼
                    ┌─────────────┐
                    │Coordination │
                    │  Service    │
                    └─────────────┘
```

**2. Local Rate Limiting (Fast Path)**

Each server maintains local token buckets:
```java
// Per-server allocation
int globalLimit = 1_000_000; // requests/second
int serverCount = 100;
int localLimit = globalLimit / serverCount; // 10,000 per server

// Local token bucket (in-memory)
TokenBucket localBucket = new TokenBucket(
    localLimit * 2,  // Capacity (allow burst)
    localLimit       // Refill rate per second
);
```

**3. Handle Uneven Distribution**

Problem: Users might be sticky to certain servers, causing uneven load.

Solution: Adaptive local limits
```java
public class AdaptiveRateLimiter {
    private volatile int localLimit;
    private final AtomicLong actualRate = new AtomicLong();
    
    // Background task adjusts limits based on actual traffic
    @Scheduled(fixedRate = 1000)
    public void adjustLimits() {
        long myRate = actualRate.getAndSet(0);
        long avgRate = coordinationService.getAverageRate();
        
        if (myRate > avgRate * 1.5) {
            // Getting more traffic, request more quota
            localLimit = coordinationService.requestQuota(localLimit * 1.2);
        } else if (myRate < avgRate * 0.5) {
            // Getting less traffic, release quota
            coordinationService.releaseQuota(localLimit * 0.2);
            localLimit *= 0.8;
        }
    }
}
```

**4. Coordination Service (Slow Path)**

```java
// Redis-based quota management
public class QuotaCoordinator {
    private static final String QUOTA_KEY = "global:quota";
    
    public int requestQuota(int requested) {
        // Lua script for atomic quota allocation
        String script = """
            local available = tonumber(redis.call('GET', KEYS[1]) or 0)
            local requested = tonumber(ARGV[1])
            local granted = math.min(available, requested)
            redis.call('DECRBY', KEYS[1], granted)
            return granted
        """;
        return redis.eval(script, QUOTA_KEY, requested);
    }
    
    // Refill quota periodically
    @Scheduled(fixedRate = 100) // Every 100ms
    public void refillQuota() {
        redis.set(QUOTA_KEY, globalLimit / 10); // 1/10th of limit per 100ms
    }
}
```

**5. Sub-millisecond Latency**

- Local bucket check: ~1 microsecond
- Only contact coordination service for quota adjustments (background)
- Use lock-free data structures

```java
public class LockFreeTokenBucket {
    private final AtomicLong tokens;
    private final AtomicLong lastRefill;
    
    public boolean tryAcquire() {
        while (true) {
            long currentTokens = tokens.get();
            long now = System.nanoTime();
            long last = lastRefill.get();
            
            // Calculate refill
            long elapsed = now - last;
            long newTokens = Math.min(capacity, 
                currentTokens + (elapsed * refillRate / 1_000_000_000));
            
            if (newTokens < 1) return false;
            
            // CAS update
            if (tokens.compareAndSet(currentTokens, newTokens - 1) &&
                lastRefill.compareAndSet(last, now)) {
                return true;
            }
            // CAS failed, retry
        }
    }
}
```

**6. Handling Edge Cases**

- **Server failure**: Coordination service redistributes quota
- **Network partition**: Servers use local limits, accept slight over-limit
- **Hot user**: Consistent hashing to spread user across servers

**7. Monitoring**

```
Metrics to track:
- Local bucket utilization per server
- Quota requests/releases per server
- Rejection rate (should be ~0 in normal operation)
- Latency percentiles (p50, p99, p99.9)
- Global vs actual request rate
```

---

## 1️⃣1️⃣ One Clean Mental Summary

Resilience patterns are your distributed system's immune system. **Rate limiting** is the bouncer at the door, controlling who gets in. **Circuit breakers** are quarantine protocols, isolating sick services before they infect others. **Bulkheads** are watertight compartments, ensuring one leak doesn't sink the ship. **Timeouts** prevent eternal waiting. **Retries** handle temporary hiccups. **Fallbacks** provide graceful degradation when nothing else works.

In production, these patterns work together: rate limiting protects the front door, bulkheads isolate dependencies, circuit breakers fail fast on prolonged outages, retries handle transient failures, timeouts prevent resource exhaustion, and fallbacks ensure users always get some response.

The key insight: **design for failure, not success**. In distributed systems, something is always failing somewhere. Resilience patterns don't prevent failures. They contain them, recover from them, and keep your system running despite them.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│                    RESILIENCE PATTERNS                       │
├─────────────────────────────────────────────────────────────┤
│ Pattern         │ Protects Against    │ Key Config          │
├─────────────────────────────────────────────────────────────┤
│ Rate Limiter    │ Traffic spikes      │ requests/second     │
│ Circuit Breaker │ Cascading failures  │ failure threshold   │
│ Bulkhead        │ Resource exhaustion │ max concurrent      │
│ Timeout         │ Slow responses      │ max wait time       │
│ Retry           │ Transient failures  │ max attempts        │
│ Fallback        │ Complete failures   │ alternative logic   │
├─────────────────────────────────────────────────────────────┤
│                    RECOMMENDED ORDER                         │
├─────────────────────────────────────────────────────────────┤
│ Request → RateLimiter → Bulkhead → Timeout →                │
│           CircuitBreaker → Retry → Service                  │
├─────────────────────────────────────────────────────────────┤
│                    KEY FORMULAS                              │
├─────────────────────────────────────────────────────────────┤
│ Exponential Backoff: delay = base × 2^attempt               │
│ Full Jitter: delay = random(0, base × 2^attempt)            │
│ Circuit Opens: failures/window >= threshold                  │
│ Bulkhead Size: peak_concurrent × 1.5 (headroom)             │
└─────────────────────────────────────────────────────────────┘
```

