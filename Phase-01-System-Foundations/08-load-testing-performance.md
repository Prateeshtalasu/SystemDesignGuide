# 🏋️ Load Testing & Performance

---

## 0️⃣ Prerequisites

Before understanding load testing, you need to know:

- **Request**: A single call from client to server.
- **Response Time/Latency**: How long a request takes (covered in Topic 3).
- **Throughput**: Requests per second the system handles (covered in Topic 3).
- **Concurrency**: Number of simultaneous users/requests.

If you understand that systems handle requests and we measure how fast and how many, you're ready.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Point

Your application works great in development with 1 user (you). But what happens with:

- 100 users?
- 1,000 users?
- 10,000 users during a flash sale?

Without testing, you're guessing. And guessing leads to:

- Crashed systems during peak traffic
- Slow response times that frustrate users
- Wasted money on over-provisioned infrastructure
- Or under-provisioned systems that can't handle growth

### What Systems Looked Like Before

Before load testing was common:

- Launch and pray
- Find out about performance issues from angry users
- Over-provision by 10x "just in case"
- No data to make capacity decisions

### What Breaks Without It

1. **Production surprises**: System crashes during peak traffic
2. **Unknown limits**: Don't know when to scale
3. **Hidden bottlenecks**: Problems only appear under load
4. **Poor user experience**: Slow responses drive users away
5. **Wasted resources**: Over-provisioning costs money

### Real Examples of the Problem

**Healthcare.gov (2013)**: Launched without adequate load testing. Expected 50,000 concurrent users, got 250,000. Site crashed repeatedly for weeks.

**Twitter's Early Days**: The "Fail Whale" appeared constantly because they didn't know their system's limits.

**Amazon Prime Day**: Amazon extensively load tests before Prime Day, simulating millions of concurrent shoppers.

---

## 2️⃣ Intuition and Mental Model

### The Highway Stress Test Analogy

Think of load testing like stress-testing a highway:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HIGHWAY STRESS TEST                                   │
│                                                                          │
│  LOAD TEST: How does the highway perform with increasing traffic?       │
│                                                                          │
│  10 cars/hour:   ════════════════════════════════════                   │
│                  Smooth flow, no issues                                  │
│                                                                          │
│  100 cars/hour:  ═══════════════════════════════════                    │
│                  Still good, slight slowdown at exits                   │
│                                                                          │
│  500 cars/hour:  ═══════════════════════════════════                    │
│                  Traffic building up, some delays                       │
│                                                                          │
│  1000 cars/hour: ═══▓▓▓═══════▓▓▓═══════▓▓▓════════                    │
│                  Congestion forming, significant delays                 │
│                                                                          │
│  2000 cars/hour: ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓                    │
│                  GRIDLOCK! Highway capacity exceeded                    │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STRESS TEST: What happens if we push beyond normal limits?             │
│                                                                          │
│  5000 cars/hour: 💥 Accidents, breakdowns, emergency response needed   │
│                  System completely fails                                │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  SPIKE TEST: Sudden rush hour traffic                                   │
│                                                                          │
│  Normal → 3000 cars in 5 minutes → Back to normal                      │
│  How quickly does the system recover?                                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key insight**: Load testing finds the point where performance degrades and where the system breaks.

---

## 3️⃣ How It Works Internally

### Types of Performance Tests

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TYPES OF PERFORMANCE TESTS                            │
│                                                                          │
│  1. LOAD TEST                                                            │
│     ─────────                                                            │
│     Purpose: Verify system handles expected load                        │
│     Pattern: Gradual increase to target load, sustain, measure          │
│                                                                          │
│     Load │                    ┌────────────────┐                        │
│          │                   ╱                  ╲                        │
│          │                  ╱                    ╲                       │
│          │                 ╱                      ╲                      │
│          │                ╱                        ╲                     │
│          │───────────────╱                          ╲───────────        │
│          └──────────────────────────────────────────────────────        │
│                         Time                                             │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  2. STRESS TEST                                                          │
│     ───────────                                                          │
│     Purpose: Find breaking point                                        │
│     Pattern: Increase load until system fails                           │
│                                                                          │
│     Load │                                          ╱ Breaking          │
│          │                                         ╱  point!            │
│          │                                        ╱                      │
│          │                                       ╱                       │
│          │                                      ╱                        │
│          │                                     ╱                         │
│          │                                    ╱                          │
│          │───────────────────────────────────╱                          │
│          └──────────────────────────────────────────────────────        │
│                         Time                                             │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  3. SPIKE TEST                                                           │
│     ──────────                                                           │
│     Purpose: Test sudden traffic bursts                                 │
│     Pattern: Normal → Sudden spike → Normal                             │
│                                                                          │
│     Load │              █████                                           │
│          │              █████                                           │
│          │              █████                                           │
│          │              █████                                           │
│          │──────────────█████──────────────                             │
│          └──────────────────────────────────────────────────────        │
│                         Time                                             │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  4. SOAK/ENDURANCE TEST                                                  │
│     ────────────────────                                                 │
│     Purpose: Find memory leaks, resource exhaustion over time           │
│     Pattern: Sustained moderate load for extended period                │
│                                                                          │
│     Load │  ┌────────────────────────────────────────────────┐          │
│          │  │                                                │          │
│          │  │          Hours or days of sustained load       │          │
│          │  │                                                │          │
│          │──┘                                                └──        │
│          └──────────────────────────────────────────────────────        │
│                         Time (hours/days)                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Metrics to Measure

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PERFORMANCE METRICS                                   │
│                                                                          │
│  LATENCY METRICS                                                         │
│  ───────────────                                                         │
│  • Response Time: Total time from request to response                   │
│  • p50 (median): 50% of requests faster than this                       │
│  • p95: 95% of requests faster than this                                │
│  • p99: 99% of requests faster than this (tail latency)                │
│  • Max: Slowest request (often an outlier)                              │
│                                                                          │
│  THROUGHPUT METRICS                                                      │
│  ─────────────────                                                       │
│  • RPS/QPS: Requests/Queries per second                                 │
│  • TPS: Transactions per second                                         │
│  • Bandwidth: Data transferred per second                               │
│                                                                          │
│  ERROR METRICS                                                           │
│  ─────────────                                                           │
│  • Error Rate: Percentage of failed requests                            │
│  • Error Types: 4xx (client), 5xx (server), timeouts                   │
│                                                                          │
│  RESOURCE METRICS                                                        │
│  ────────────────                                                        │
│  • CPU Utilization: Percentage of CPU used                              │
│  • Memory Usage: RAM consumption                                        │
│  • Disk I/O: Read/write operations                                      │
│  • Network I/O: Bandwidth usage                                         │
│  • Thread Pool: Active vs available threads                             │
│  • Connection Pool: Active vs available connections                     │
│                                                                          │
│  SATURATION METRICS                                                      │
│  ──────────────────                                                      │
│  • Queue Depth: Requests waiting to be processed                        │
│  • Thread Pool Exhaustion: All threads busy                             │
│  • Connection Pool Exhaustion: All connections in use                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### QPS Calculations

**Back-of-envelope calculation for capacity planning**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    QPS CALCULATION                                       │
│                                                                          │
│  Scenario: E-commerce site planning for Black Friday                    │
│                                                                          │
│  Given:                                                                  │
│  • Expected users: 1,000,000 during peak hour                           │
│  • Average session: 30 minutes                                          │
│  • Pages per session: 20                                                │
│  • API calls per page: 5                                                │
│                                                                          │
│  Calculation:                                                            │
│                                                                          │
│  Concurrent users = 1,000,000 × (30/60) = 500,000                       │
│  (Users online at any moment)                                           │
│                                                                          │
│  Page views per second = 500,000 × 20 / (30 × 60) = 5,556 pages/sec    │
│                                                                          │
│  API calls per second = 5,556 × 5 = 27,778 QPS                         │
│                                                                          │
│  Add 2x safety margin: 55,556 QPS target                                │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  Server capacity calculation:                                            │
│                                                                          │
│  If each server handles 1,000 QPS:                                      │
│  Servers needed = 55,556 / 1,000 = 56 servers                          │
│                                                                          │
│  Add redundancy (N+2): 58 servers minimum                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Bottleneck Identification

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON BOTTLENECKS                                    │
│                                                                          │
│  1. CPU BOUND                                                            │
│     Symptoms: CPU at 100%, response time increases with load           │
│     Causes: Complex calculations, inefficient algorithms                │
│     Solutions: Optimize code, add more CPU, cache results               │
│                                                                          │
│  2. MEMORY BOUND                                                         │
│     Symptoms: High memory usage, GC pauses, OOM errors                  │
│     Causes: Memory leaks, large objects, too much caching              │
│     Solutions: Fix leaks, tune GC, add memory, reduce object size      │
│                                                                          │
│  3. I/O BOUND (Database)                                                │
│     Symptoms: High DB latency, connection pool exhaustion              │
│     Causes: Slow queries, missing indexes, too many queries            │
│     Solutions: Optimize queries, add indexes, caching, read replicas   │
│                                                                          │
│  4. I/O BOUND (Disk)                                                    │
│     Symptoms: High disk wait time, slow file operations                │
│     Causes: Logging too much, file-based sessions, temp files          │
│     Solutions: Async I/O, SSD, reduce logging, move to memory          │
│                                                                          │
│  5. NETWORK BOUND                                                        │
│     Symptoms: High network latency, bandwidth saturation               │
│     Causes: Large payloads, chatty protocols, external API calls       │
│     Solutions: Compress data, batch requests, CDN, connection pooling  │
│                                                                          │
│  6. THREAD POOL EXHAUSTION                                               │
│     Symptoms: Requests queuing, timeouts, thread pool at max           │
│     Causes: Slow downstream services, blocking I/O                     │
│     Solutions: Async processing, increase pool, fix slow dependencies  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Simulation-First Explanation

### Load Test Execution: Step by Step

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LOAD TEST EXECUTION                                   │
│                                                                          │
│  Target: Test API endpoint /api/products                                │
│  Goal: Find maximum throughput while maintaining p99 < 500ms            │
│                                                                          │
│  PHASE 1: Baseline (5 minutes)                                          │
│  ────────────────────────────                                           │
│  Load: 10 requests/second                                               │
│  Results:                                                                │
│  • p50: 45ms                                                            │
│  • p99: 120ms                                                           │
│  • Error rate: 0%                                                       │
│  • CPU: 5%                                                              │
│  Conclusion: System handles baseline easily ✓                           │
│                                                                          │
│  PHASE 2: Ramp Up (10 minutes)                                          │
│  ───────────────────────────                                            │
│  Load: 10 → 100 → 500 → 1000 requests/second                           │
│                                                                          │
│  At 100 RPS:                                                             │
│  • p50: 48ms, p99: 150ms, CPU: 15% ✓                                   │
│                                                                          │
│  At 500 RPS:                                                             │
│  • p50: 55ms, p99: 280ms, CPU: 45% ✓                                   │
│                                                                          │
│  At 1000 RPS:                                                            │
│  • p50: 85ms, p99: 450ms, CPU: 78% ✓ (close to limit)                  │
│                                                                          │
│  PHASE 3: Stress (5 minutes)                                            │
│  ──────────────────────────                                             │
│  Load: 1500 requests/second                                             │
│  Results:                                                                │
│  • p50: 250ms ⚠️                                                        │
│  • p99: 2500ms ❌ (exceeds 500ms target)                                │
│  • Error rate: 5% ❌                                                     │
│  • CPU: 98%                                                              │
│  Conclusion: System breaking at 1500 RPS                                │
│                                                                          │
│  PHASE 4: Find Sweet Spot                                               │
│  ────────────────────────                                               │
│  Load: 1200 requests/second                                             │
│  Results:                                                                │
│  • p50: 95ms                                                            │
│  • p99: 480ms ✓ (just under 500ms)                                     │
│  • Error rate: 0.1%                                                     │
│  • CPU: 85%                                                              │
│                                                                          │
│  CONCLUSION:                                                             │
│  Maximum sustainable load: ~1200 RPS                                    │
│  Recommended operating load: 1000 RPS (80% of max for safety)          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Identifying the Bottleneck

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BOTTLENECK INVESTIGATION                              │
│                                                                          │
│  Observation: At 1500 RPS, p99 latency spikes to 2500ms                │
│                                                                          │
│  Step 1: Check application metrics                                      │
│  ───────────────────────────────                                        │
│  • CPU: 98% → High but not 100%                                        │
│  • Memory: 60% → OK                                                     │
│  • Thread pool: 200/200 → EXHAUSTED! ❌                                 │
│                                                                          │
│  Hypothesis: Thread pool exhaustion causing queuing                     │
│                                                                          │
│  Step 2: Check what threads are doing                                   │
│  ────────────────────────────────────                                   │
│  Thread dump shows:                                                      │
│  • 180 threads waiting for database connection                          │
│  • 20 threads processing requests                                       │
│                                                                          │
│  Step 3: Check database                                                  │
│  ─────────────────────────                                              │
│  • Connection pool: 20/20 → EXHAUSTED! ❌                               │
│  • Query time: 150ms average                                            │
│  • Slow query log: SELECT * FROM products WHERE... (no index!)         │
│                                                                          │
│  ROOT CAUSE FOUND:                                                       │
│  1. Database query missing index (150ms instead of 5ms)                │
│  2. DB connection pool too small (20 connections)                       │
│  3. App thread pool waiting for DB connections                          │
│                                                                          │
│  FIXES:                                                                  │
│  1. Add index: CREATE INDEX idx_products_category ON products(category)│
│  2. Increase DB connection pool: 20 → 50                                │
│  3. Add query result caching                                            │
│                                                                          │
│  AFTER FIXES:                                                            │
│  At 1500 RPS:                                                            │
│  • p50: 35ms, p99: 180ms ✓                                             │
│  • Error rate: 0%                                                       │
│  • CPU: 55%                                                              │
│  New maximum: ~3000 RPS                                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Real Systems at Real Companies

**Amazon**:

- Load tests before every Prime Day
- Simulates 10x expected traffic
- Tests every microservice independently
- Uses "GameDay" exercises to simulate failures under load

**Netflix**:

- Continuous performance testing in production
- Uses Chaos Engineering combined with load testing
- Tests can run for days to find memory leaks
- Region evacuation tests under full load

**Google**:

- DiRT (Disaster Recovery Testing) includes load testing
- Tests at global scale
- Automated performance regression detection

### Performance Testing Tools

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    POPULAR LOAD TESTING TOOLS                            │
│                                                                          │
│  JMeter (Apache)                                                         │
│  ───────────────                                                         │
│  • Java-based, GUI and CLI                                              │
│  • Supports HTTP, JDBC, JMS, etc.                                       │
│  • Good for complex scenarios                                           │
│  • Can be resource-heavy                                                │
│  • Best for: Traditional load testing                                   │
│                                                                          │
│  Gatling                                                                 │
│  ───────                                                                 │
│  • Scala-based, code-as-tests                                           │
│  • Excellent reports                                                     │
│  • Efficient resource usage                                             │
│  • Best for: CI/CD integration, developers                              │
│                                                                          │
│  k6 (Grafana)                                                            │
│  ────────────                                                            │
│  • JavaScript-based                                                      │
│  • Modern, developer-friendly                                           │
│  • Cloud and local execution                                            │
│  • Best for: Modern teams, cloud-native                                 │
│                                                                          │
│  Locust                                                                  │
│  ──────                                                                  │
│  • Python-based                                                          │
│  • Distributed testing                                                   │
│  • Real-time web UI                                                      │
│  • Best for: Python teams, custom scenarios                             │
│                                                                          │
│  wrk / wrk2                                                              │
│  ─────────                                                               │
│  • C-based, very lightweight                                            │
│  • Extremely high throughput                                            │
│  • Lua scripting                                                         │
│  • Best for: Quick benchmarks, high RPS testing                         │
│                                                                          │
│  Apache Bench (ab)                                                       │
│  ─────────────────                                                       │
│  • Simple command-line tool                                             │
│  • Single URL testing                                                    │
│  • Best for: Quick sanity checks                                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6️⃣ How to Implement Load Testing

### Using k6 (Modern Approach)

```javascript
// load-test.js
import http from "k6/http";
import { check, sleep } from "k6";
import { Rate, Trend } from "k6/metrics";

// Custom metrics
const errorRate = new Rate("errors");
const productLatency = new Trend("product_latency");

// Test configuration
export const options = {
  // Stages define the load pattern
  stages: [
    { duration: "2m", target: 100 }, // Ramp up to 100 users over 2 min
    { duration: "5m", target: 100 }, // Stay at 100 users for 5 min
    { duration: "2m", target: 500 }, // Ramp up to 500 users
    { duration: "5m", target: 500 }, // Stay at 500 users
    { duration: "2m", target: 1000 }, // Ramp up to 1000 users
    { duration: "5m", target: 1000 }, // Stay at 1000 users
    { duration: "2m", target: 0 }, // Ramp down to 0
  ],

  // Thresholds define pass/fail criteria
  thresholds: {
    http_req_duration: ["p(95)<500", "p(99)<1000"], // 95% < 500ms, 99% < 1s
    errors: ["rate<0.01"], // Error rate < 1%
    product_latency: ["p(95)<300"], // Product API < 300ms
  },
};

// Setup: Run once before test
export function setup() {
  // Login and get auth token
  const loginRes = http.post("http://api.example.com/auth/login", {
    username: "testuser",
    password: "testpass",
  });

  return { authToken: loginRes.json("token") };
}

// Main test function: Run for each virtual user
export default function (data) {
  const headers = {
    Authorization: `Bearer ${data.authToken}`,
    "Content-Type": "application/json",
  };

  // Scenario 1: Browse products (60% of traffic)
  if (Math.random() < 0.6) {
    const start = Date.now();
    const res = http.get("http://api.example.com/products", { headers });
    productLatency.add(Date.now() - start);

    check(res, {
      "products status 200": (r) => r.status === 200,
      "products has items": (r) => r.json("items").length > 0,
    });

    errorRate.add(res.status !== 200);
  }

  // Scenario 2: View product detail (30% of traffic)
  else if (Math.random() < 0.9) {
    const productId = Math.floor(Math.random() * 1000) + 1;
    const res = http.get(`http://api.example.com/products/${productId}`, {
      headers,
    });

    check(res, {
      "product detail status 200": (r) => r.status === 200,
    });

    errorRate.add(res.status !== 200);
  }

  // Scenario 3: Add to cart (10% of traffic)
  else {
    const payload = JSON.stringify({
      productId: Math.floor(Math.random() * 1000) + 1,
      quantity: 1,
    });

    const res = http.post("http://api.example.com/cart/add", payload, {
      headers,
    });

    check(res, {
      "add to cart status 200": (r) => r.status === 200 || r.status === 201,
    });

    errorRate.add(res.status >= 400);
  }

  // Think time: Simulate real user behavior
  sleep(Math.random() * 3 + 1); // 1-4 seconds between requests
}

// Teardown: Run once after test
export function teardown(data) {
  // Cleanup: logout, delete test data, etc.
  http.post("http://api.example.com/auth/logout", null, {
    headers: { Authorization: `Bearer ${data.authToken}` },
  });
}
```

### Using Gatling (Scala-based)

```scala
// ProductLoadTest.scala
package com.example.loadtest

import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class ProductLoadTest extends Simulation {

  // HTTP configuration
  val httpProtocol = http
    .baseUrl("http://api.example.com")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
    .userAgentHeader("Gatling Load Test")

  // Feeder for random product IDs
  val productIdFeeder = Iterator.continually(
    Map("productId" -> (scala.util.Random.nextInt(1000) + 1))
  )

  // Scenario: Browse products
  val browseProducts = scenario("Browse Products")
    .exec(
      http("Get Products")
        .get("/products")
        .queryParam("page", "1")
        .queryParam("limit", "20")
        .check(status.is(200))
        .check(jsonPath("$.items").exists)
    )
    .pause(1.second, 3.seconds)  // Think time

  // Scenario: View product details
  val viewProduct = scenario("View Product")
    .feed(productIdFeeder)
    .exec(
      http("Get Product Detail")
        .get("/products/${productId}")
        .check(status.is(200))
        .check(jsonPath("$.id").exists)
    )
    .pause(2.seconds, 5.seconds)

  // Scenario: Add to cart
  val addToCart = scenario("Add to Cart")
    .feed(productIdFeeder)
    .exec(
      http("Add to Cart")
        .post("/cart/add")
        .body(StringBody("""{"productId": ${productId}, "quantity": 1}"""))
        .check(status.in(200, 201))
    )
    .pause(1.second)

  // Load profile
  setUp(
    browseProducts.inject(
      rampUsers(100).during(2.minutes),    // Ramp to 100 users
      constantUsersPerSec(50).during(5.minutes)  // 50 users/sec for 5 min
    ),
    viewProduct.inject(
      rampUsers(200).during(2.minutes),
      constantUsersPerSec(100).during(5.minutes)
    ),
    addToCart.inject(
      rampUsers(50).during(2.minutes),
      constantUsersPerSec(20).during(5.minutes)
    )
  ).protocols(httpProtocol)
   .assertions(
     global.responseTime.percentile(95).lt(500),  // p95 < 500ms
     global.successfulRequests.percent.gt(99)     // > 99% success
   )
}
```

### Using JMeter (Configuration)

```xml
<!-- jmeter-test-plan.jmx (simplified) -->
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="Product API Load Test">
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments"/>
    </TestPlan>
    <hashTree>
      <!-- Thread Group: Simulates users -->
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Users">
        <intProp name="ThreadGroup.num_threads">100</intProp>
        <intProp name="ThreadGroup.ramp_time">60</intProp>
        <boolProp name="ThreadGroup.scheduler">true</boolProp>
        <stringProp name="ThreadGroup.duration">300</stringProp>
      </ThreadGroup>
      <hashTree>
        <!-- HTTP Request -->
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="Get Products">
          <stringProp name="HTTPSampler.domain">api.example.com</stringProp>
          <stringProp name="HTTPSampler.port">443</stringProp>
          <stringProp name="HTTPSampler.protocol">https</stringProp>
          <stringProp name="HTTPSampler.path">/products</stringProp>
          <stringProp name="HTTPSampler.method">GET</stringProp>
        </HTTPSamplerProxy>
        <hashTree>
          <!-- Response Assertion -->
          <ResponseAssertion guiclass="AssertionGui" testclass="ResponseAssertion" testname="Response Assertion">
            <collectionProp name="Asserion.test_strings">
              <stringProp>200</stringProp>
            </collectionProp>
            <intProp name="Assertion.test_type">8</intProp>
            <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
          </ResponseAssertion>
        </hashTree>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

### Running Load Tests

```bash
# k6
k6 run load-test.js
k6 run --vus 100 --duration 5m load-test.js  # Quick test

# Gatling
./gradlew gatlingRun-com.example.loadtest.ProductLoadTest

# JMeter (CLI mode for CI/CD)
jmeter -n -t test-plan.jmx -l results.jtl -e -o report/

# Apache Bench (quick sanity check)
ab -n 10000 -c 100 http://api.example.com/products

# wrk (high throughput testing)
wrk -t12 -c400 -d30s http://api.example.com/products
```

### Spring Boot Performance Configuration

```yaml
# application.yml - Performance tuning
server:
  tomcat:
    threads:
      max: 200 # Max worker threads
      min-spare: 20 # Min idle threads
    max-connections: 10000
    accept-count: 100 # Request queue size
    connection-timeout: 20000

spring:
  datasource:
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000

  redis:
    lettuce:
      pool:
        max-active: 50
        max-idle: 20
        min-idle: 5

# Enable response compression
server.compression:
  enabled: true
  mime-types: application/json,application/xml,text/html,text/plain
  min-response-size: 1024

# Actuator for metrics during load test
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

**1. Testing from single machine**

```
WRONG: Run load test from your laptop
       - Limited network bandwidth
       - Single source IP might be rate-limited
       - Can't generate enough load

RIGHT: Use distributed load generation
       - Multiple machines/regions
       - Cloud-based load testing (k6 Cloud, Gatling Enterprise)
       - Realistic geographic distribution
```

**2. No think time**

```
WRONG: Fire requests as fast as possible
       - Unrealistic traffic pattern
       - Doesn't simulate real user behavior

RIGHT: Add realistic delays between requests
       - Users read pages, fill forms, think
       - Typically 1-10 seconds between actions
```

**3. Testing only happy path**

```
WRONG: Only test successful scenarios
       - Miss error handling performance
       - Miss cache miss scenarios

RIGHT: Include realistic scenarios
       - 404 errors (product not found)
       - Cache misses (first-time visitors)
       - Slow database queries
       - Large responses
```

**4. Ignoring coordinated omission**

```
WRONG: Send next request only after previous completes
       - If server is slow, you send fewer requests
       - Hides true latency under load

RIGHT: Send requests at fixed rate regardless of response
       - Reveals true latency when server is overloaded
       - Use tools that handle this (wrk2, k6)
```

### Test Environment Considerations

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TEST ENVIRONMENT                                      │
│                                                                          │
│  Production-like environment is CRITICAL                                │
│                                                                          │
│  Differences that invalidate results:                                   │
│  ❌ Smaller database (10K rows vs 10M rows)                             │
│  ❌ No CDN in test environment                                          │
│  ❌ Single instance vs clustered                                        │
│  ❌ Different hardware (CPU, memory, disk)                              │
│  ❌ No network latency (same datacenter)                                │
│  ❌ Empty caches                                                         │
│                                                                          │
│  Minimum requirements for valid load testing:                           │
│  ✓ Same database size (or representative sample)                       │
│  ✓ Same architecture (load balancer, multiple instances)               │
│  ✓ Similar hardware specs                                               │
│  ✓ Warmed caches                                                        │
│  ✓ Realistic data distribution                                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8️⃣ When NOT to Load Test

### Situations Where It's Less Critical

1. **Internal tools with few users**: If only 10 people use it, basic testing is enough
2. **Prototype/MVP**: Ship first, optimize later
3. **Read-heavy static sites**: CDN handles the load
4. **Already in production**: Use production metrics instead (with caution)

### When to Use Production Testing Instead

- **Canary deployments**: Test with real traffic
- **Feature flags**: Gradual rollout with monitoring
- **Shadow traffic**: Mirror production requests to test environment

---

## 9️⃣ Comparison: Load Testing Approaches

| Approach             | Pros                   | Cons                    | Best For       |
| -------------------- | ---------------------- | ----------------------- | -------------- |
| Synthetic (pre-prod) | Controlled, repeatable | May not reflect reality | Finding limits |
| Production (canary)  | Real traffic patterns  | Risk to users           | Validation     |
| Shadow traffic       | Real patterns, no risk | Complex setup           | Large systems  |
| Chaos + Load         | Tests resilience       | Complex                 | Mature systems |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is the difference between load testing and stress testing?**

A: Load testing verifies the system handles expected load. You ramp up to your target (say, 1000 users) and measure if response times and error rates stay acceptable. Stress testing finds the breaking point. You keep increasing load until the system fails, to understand its limits. Load testing asks "can we handle normal traffic?" Stress testing asks "how much can we handle before breaking?"

**Q: What metrics would you measure in a load test?**

A: Key metrics are: (1) Latency: response time percentiles (p50, p95, p99), not just average. (2) Throughput: requests per second successfully processed. (3) Error rate: percentage of failed requests. (4) Resource utilization: CPU, memory, disk I/O, network. (5) Saturation: queue depths, thread pool usage, connection pool usage. I'd set thresholds for each, like "p99 latency must be under 500ms" and "error rate must be under 1%."

### L5 (Mid-Level) Questions

**Q: How would you identify a performance bottleneck?**

A: I'd follow a systematic approach: (1) Start with the metrics that are out of bounds (high latency, errors). (2) Check resource utilization: is CPU, memory, or I/O maxed out? (3) If CPU is high, profile the code to find hot spots. (4) If CPU is low but latency is high, check for blocking: thread dumps show what threads are waiting for. (5) Check external dependencies: database query times, external API latencies. (6) Check for resource exhaustion: connection pools, thread pools. (7) Use distributed tracing to see where time is spent in the request flow. Usually it's the database (missing indexes, N+1 queries) or a slow external dependency.

**Q: How do you calculate required capacity for a system?**

A: I use back-of-envelope calculations: (1) Estimate peak concurrent users from expected total users and session duration. (2) Calculate requests per second from pages per session and API calls per page. (3) Add safety margin (usually 2x). (4) Divide by per-server capacity (from load testing) to get server count. (5) Add redundancy (N+2 for high availability). For example: 1M peak users, 30-min sessions = 500K concurrent. 20 pages/session, 5 API calls/page = 28K QPS. With 2x margin = 56K QPS. If each server handles 1K QPS, need 56 servers + 2 for redundancy = 58 servers.

### L6 (Senior) Questions

**Q: How would you set up continuous performance testing in a CI/CD pipeline?**

A: I'd implement multiple levels: (1) Unit performance tests: Micro-benchmarks for critical algorithms, run on every commit. (2) Integration performance tests: Test key API endpoints with moderate load (100 users, 2 minutes), run on every PR. Fail if latency regresses by >10%. (3) Full load tests: Complete scenarios with realistic load, run nightly or before release. (4) Automated analysis: Compare results against baseline, alert on regressions. (5) Store historical data: Track performance trends over time. Tools: k6 or Gatling for tests, Prometheus for metrics, Grafana for visualization, custom scripts for regression detection. Key is making it automated and integrated so performance issues are caught early.

**Q: How would you load test a microservices architecture?**

A: Microservices add complexity: (1) Test individual services first: Understand each service's limits in isolation. (2) Test end-to-end flows: User journeys that span multiple services. (3) Identify the weakest link: The slowest service limits the whole flow. (4) Test with realistic service mesh behavior: Include retries, circuit breakers, timeouts. (5) Use distributed tracing: See where latency accumulates across services. (6) Test failure scenarios: What happens when one service is slow or down? (7) Consider cascading effects: One slow service can exhaust thread pools in callers. I'd also test the infrastructure: service discovery, load balancers, message queues. The goal is understanding system behavior, not just individual component performance.

---

## 1️⃣1️⃣ One Clean Mental Summary

Load testing answers "how much can my system handle?" before users find out the hard way. Run load tests to verify expected capacity, stress tests to find breaking points, and spike tests for sudden traffic bursts. Key metrics are latency percentiles (not averages), throughput, error rates, and resource utilization. Common bottlenecks are database queries (add indexes, caching), thread pool exhaustion (async processing, increase pools), and external dependencies (timeouts, circuit breakers). Test in production-like environments with realistic data and think times. The goal isn't just finding limits, but understanding system behavior under load so you can make informed capacity decisions.
