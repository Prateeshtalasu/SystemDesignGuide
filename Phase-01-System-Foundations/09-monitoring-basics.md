# 📡 Monitoring Basics: Observability for Production Systems

---

## 0️⃣ Prerequisites

Before understanding monitoring, you need to know:

- **Production System**: An application running and serving real users.
- **Metrics**: Numerical measurements of system behavior (covered in Topic 3).
- **Server/Service**: A running application that handles requests.
- **Log**: A record of events that happened in the system.

If you understand that systems produce data about their behavior and we need to watch that data, you're ready.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Point

Your application is running in production. But:

- Is it healthy right now?
- Are users experiencing errors?
- Is it about to run out of memory?
- Which request is causing the slowdown?
- Did the last deployment break something?

Without monitoring, you're blind. You only find out about problems when users complain, or worse, when revenue drops.

### What Systems Looked Like Before

Before modern monitoring:

- Check server manually via SSH
- Wait for user complaints
- Look at logs only after incidents
- No historical data to compare against
- "It works on my machine" syndrome

### What Breaks Without It

1. **Delayed incident detection**: Problems exist for hours before anyone notices
2. **Blind troubleshooting**: No data to diagnose issues
3. **No capacity planning**: Don't know when to scale
4. **No accountability**: Can't measure SLOs
5. **Repeated incidents**: Can't learn from past failures

### Real Examples of the Problem

**Knight Capital (2012)**: A deployment issue caused $440M loss in 45 minutes. Better monitoring could have detected the anomaly in seconds.

**GitLab (2017)**: Accidentally deleted production database. Realized monitoring showed backup jobs had been failing for days, but no one was watching.

---

## 2️⃣ Intuition and Mental Model

### The Car Dashboard Analogy

Think of monitoring like a car dashboard:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CAR DASHBOARD ANALOGY                                 │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                     CAR DASHBOARD                                │    │
│  │                                                                  │    │
│  │   ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐                 │    │
│  │   │ Fuel │    │Speed │    │ RPM  │    │ Temp │                 │    │
│  │   │ ████ │    │ 65   │    │ 3000 │    │ ░░█░ │                 │    │
│  │   │ 75%  │    │ mph  │    │      │    │ OK   │                 │    │
│  │   └──────┘    └──────┘    └──────┘    └──────┘                 │    │
│  │                                                                  │    │
│  │   🔴 Check Engine    ⚠️ Low Tire Pressure                       │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  METRICS = Gauges (speed, fuel, RPM, temperature)                       │
│  ALERTS = Warning lights (check engine, low tire)                       │
│  LOGS = Trip computer history (last 10 trips, fuel economy)            │
│  TRACES = GPS route tracking (how you got from A to B)                 │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  Without a dashboard:                                                    │
│  - You'd run out of fuel unexpectedly                                   │
│  - You'd overheat the engine                                            │
│  - You'd get speeding tickets                                           │
│  - You'd miss warning signs of problems                                 │
│                                                                          │
│  Same with software: without monitoring, you're driving blind           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key insight**: Monitoring gives you visibility into what's happening inside your system.

---

## 3️⃣ How It Works Internally

### The Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THREE PILLARS OF OBSERVABILITY                        │
│                                                                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐         │
│  │     METRICS     │  │      LOGS       │  │     TRACES      │         │
│  │                 │  │                 │  │                 │         │
│  │  Numerical      │  │  Textual        │  │  Request flow   │         │
│  │  measurements   │  │  records of     │  │  across         │         │
│  │  over time      │  │  events         │  │  services       │         │
│  │                 │  │                 │  │                 │         │
│  │  "How much?"    │  │  "What          │  │  "What path?"   │         │
│  │  "How fast?"    │  │   happened?"    │  │  "Where slow?"  │         │
│  │                 │  │                 │  │                 │         │
│  │  Examples:      │  │  Examples:      │  │  Examples:      │         │
│  │  - CPU: 75%     │  │  - Error msg    │  │  - Request ID   │         │
│  │  - Latency: 50ms│  │  - User login   │  │  - Service hops │         │
│  │  - Requests: 1K │  │  - Stack trace  │  │  - Timing spans │         │
│  │                 │  │                 │  │                 │         │
│  │  Tools:         │  │  Tools:         │  │  Tools:         │         │
│  │  - Prometheus   │  │  - ELK Stack    │  │  - Jaeger       │         │
│  │  - Datadog      │  │  - Splunk       │  │  - Zipkin       │         │
│  │  - CloudWatch   │  │  - Loki         │  │  - Datadog APM  │         │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘         │
│                                                                          │
│  Together they answer:                                                   │
│  - Metrics: "Is there a problem?" (high-level)                          │
│  - Logs: "What exactly happened?" (detailed)                            │
│  - Traces: "Where in the system?" (distributed)                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Metrics: What to Monitor

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE FOUR GOLDEN SIGNALS                               │
│                    (Google SRE's Framework)                              │
│                                                                          │
│  1. LATENCY                                                              │
│     ─────────                                                            │
│     The time it takes to service a request                              │
│                                                                          │
│     Measure:                                                             │
│     • Successful request latency                                        │
│     • Failed request latency (often different!)                         │
│     • Percentiles: p50, p90, p95, p99                                  │
│                                                                          │
│     Alert when: p99 > 500ms                                             │
│                                                                          │
│  2. TRAFFIC                                                              │
│     ───────                                                              │
│     How much demand is being placed on your system                      │
│                                                                          │
│     Measure:                                                             │
│     • Requests per second (RPS)                                         │
│     • Transactions per second                                           │
│     • Concurrent users                                                   │
│                                                                          │
│     Alert when: Traffic drops suddenly (might indicate problem)         │
│                                                                          │
│  3. ERRORS                                                               │
│     ──────                                                               │
│     The rate of requests that fail                                      │
│                                                                          │
│     Measure:                                                             │
│     • HTTP 5xx rate (server errors)                                     │
│     • HTTP 4xx rate (client errors)                                     │
│     • Application exceptions                                            │
│     • Failed health checks                                              │
│                                                                          │
│     Alert when: Error rate > 1%                                         │
│                                                                          │
│  4. SATURATION                                                           │
│     ──────────                                                           │
│     How "full" your service is                                          │
│                                                                          │
│     Measure:                                                             │
│     • CPU utilization                                                    │
│     • Memory utilization                                                 │
│     • Disk I/O utilization                                              │
│     • Thread pool usage                                                  │
│     • Connection pool usage                                             │
│                                                                          │
│     Alert when: CPU > 80% for 5 minutes                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### USE Method (For Resources)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    USE METHOD                                            │
│                    (Brendan Gregg's Framework)                           │
│                                                                          │
│  For every resource (CPU, memory, disk, network):                       │
│                                                                          │
│  U - UTILIZATION                                                         │
│      How busy is the resource?                                          │
│      Example: CPU at 75%                                                │
│                                                                          │
│  S - SATURATION                                                          │
│      How much extra work is queued?                                     │
│      Example: 10 requests waiting in queue                              │
│                                                                          │
│  E - ERRORS                                                              │
│      How many errors occurred?                                          │
│      Example: 5 disk I/O errors                                         │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ Resource   │ Utilization      │ Saturation    │ Errors           │   │
│  │────────────────────────────────────────────────────────────────│   │
│  │ CPU        │ CPU %            │ Run queue     │ -                │   │
│  │ Memory     │ Used/Total       │ Swap usage    │ OOM events       │   │
│  │ Disk       │ Disk busy %      │ I/O queue     │ I/O errors       │   │
│  │ Network    │ Bandwidth used   │ Socket queue  │ Packet drops     │   │
│  │ Threads    │ Active/Max       │ Queue depth   │ Rejections       │   │
│  │ DB Conns   │ Used/Max         │ Wait time     │ Timeouts         │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Logs: What to Log

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LOGGING BEST PRACTICES                                │
│                                                                          │
│  LOG LEVELS:                                                             │
│  ───────────                                                             │
│  ERROR:   Something failed, needs attention                             │
│           "Payment processing failed for order 123"                     │
│                                                                          │
│  WARN:    Something unexpected but handled                              │
│           "Retry succeeded after 2 attempts"                            │
│                                                                          │
│  INFO:    Important business events                                      │
│           "Order 123 created for customer 456"                          │
│                                                                          │
│  DEBUG:   Detailed technical information                                │
│           "Database query took 45ms"                                    │
│           (Usually disabled in production)                              │
│                                                                          │
│  TRACE:   Very detailed, step-by-step                                   │
│           "Entering method processPayment()"                            │
│           (Rarely used in production)                                   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  WHAT TO INCLUDE IN LOGS:                                               │
│  ─────────────────────────                                              │
│  ✓ Timestamp (ISO 8601: 2024-12-23T10:30:00.123Z)                      │
│  ✓ Log level (ERROR, WARN, INFO)                                       │
│  ✓ Service name (order-service)                                        │
│  ✓ Request/Trace ID (for correlation)                                  │
│  ✓ User ID (for debugging user issues)                                 │
│  ✓ Message (human-readable)                                            │
│  ✓ Structured data (JSON for machine parsing)                          │
│                                                                          │
│  WHAT NOT TO LOG:                                                        │
│  ────────────────                                                        │
│  ✗ Passwords, tokens, secrets                                          │
│  ✗ Credit card numbers, SSN                                            │
│  ✗ Personal health information                                         │
│  ✗ Full request/response bodies (too verbose)                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Traces: Distributed Tracing

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DISTRIBUTED TRACING                                   │
│                                                                          │
│  A trace follows a request across multiple services                     │
│                                                                          │
│  Request: GET /api/orders/123                                           │
│  Trace ID: abc-123-def                                                  │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ Time ──────────────────────────────────────────────────────────►│    │
│  │                                                                  │    │
│  │ API Gateway      ████████████████████████████████████  150ms    │    │
│  │   │                                                              │    │
│  │   └─► Order Svc    ██████████████████████████████  120ms        │    │
│  │         │                                                        │    │
│  │         ├─► User Svc     ████████  20ms                         │    │
│  │         │                                                        │    │
│  │         ├─► Inventory    ██████████████  40ms                   │    │
│  │         │                                                        │    │
│  │         └─► Database         ████████████████████  50ms         │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Each box is a "span":                                                  │
│  • Span ID: Unique identifier                                           │
│  • Parent Span ID: Who called this                                      │
│  • Service name: Which service                                          │
│  • Operation: What it did                                               │
│  • Duration: How long it took                                           │
│  • Tags: Additional context (user_id, error, etc.)                     │
│                                                                          │
│  From this trace we can see:                                            │
│  • Database is the slowest (50ms)                                       │
│  • Total request time is 150ms                                          │
│  • Order service waits for 3 downstream calls                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Health Checks

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HEALTH CHECK TYPES                                    │
│                                                                          │
│  1. LIVENESS CHECK                                                       │
│     ─────────────────                                                    │
│     "Is the application running?"                                       │
│     If fails: Restart the container/process                             │
│                                                                          │
│     Example: GET /health/live                                           │
│     Response: 200 OK (just proves process is alive)                     │
│                                                                          │
│  2. READINESS CHECK                                                      │
│     ──────────────────                                                   │
│     "Can the application handle requests?"                              │
│     If fails: Remove from load balancer (don't restart)                │
│                                                                          │
│     Example: GET /health/ready                                          │
│     Checks: Database connection, cache connection, dependencies         │
│     Response: 200 OK or 503 Service Unavailable                        │
│                                                                          │
│  3. STARTUP CHECK                                                        │
│     ─────────────────                                                    │
│     "Has the application finished starting?"                            │
│     Used for slow-starting applications                                 │
│                                                                          │
│     Example: GET /health/startup                                        │
│     Checks: Migrations complete, caches warmed                          │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  DEEP HEALTH CHECK (for debugging)                                      │
│  ─────────────────────────────────                                      │
│  GET /health/details                                                    │
│                                                                          │
│  {                                                                       │
│    "status": "UP",                                                       │
│    "components": {                                                       │
│      "database": {"status": "UP", "latency_ms": 5},                    │
│      "redis": {"status": "UP", "latency_ms": 2},                       │
│      "payment_api": {"status": "DOWN", "error": "Connection refused"}, │
│      "disk": {"status": "UP", "free_gb": 45}                           │
│    }                                                                     │
│  }                                                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Alerting Basics

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ALERTING PRINCIPLES                                   │
│                                                                          │
│  GOOD ALERTS:                                                            │
│  ────────────                                                            │
│  ✓ Actionable: Someone can do something about it                       │
│  ✓ Urgent: Needs attention now (or soon)                               │
│  ✓ Clear: What's wrong and what to do                                  │
│  ✓ Rare: Not crying wolf constantly                                    │
│                                                                          │
│  BAD ALERTS:                                                             │
│  ───────────                                                             │
│  ✗ "CPU is at 75%" (So what? Is anything broken?)                      │
│  ✗ "Disk usage is 60%" (Not urgent, not actionable now)                │
│  ✗ "Error occurred" (Which error? Where? Impact?)                      │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  ALERT SEVERITY LEVELS:                                                  │
│  ──────────────────────                                                  │
│                                                                          │
│  P1 - CRITICAL (Page immediately, 24/7)                                 │
│       • Service completely down                                         │
│       • Data loss occurring                                             │
│       • Security breach                                                  │
│       Response: Immediate (minutes)                                     │
│                                                                          │
│  P2 - HIGH (Page during business hours)                                 │
│       • Service degraded significantly                                  │
│       • Error rate > 5%                                                 │
│       Response: Within 1 hour                                           │
│                                                                          │
│  P3 - MEDIUM (Ticket, next business day)                                │
│       • Performance degradation                                         │
│       • Non-critical feature broken                                     │
│       Response: Within 24 hours                                         │
│                                                                          │
│  P4 - LOW (Ticket, this sprint)                                         │
│       • Minor issues                                                     │
│       • Optimization opportunities                                      │
│       Response: Within 1 week                                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Simulation-First Explanation

### Setting Up Monitoring: Step by Step

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MONITORING SETUP FLOW                                 │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    YOUR APPLICATION                              │    │
│  │                                                                  │    │
│  │  1. Instrument code (add metrics, logs, traces)                 │    │
│  │                                                                  │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                         │    │
│  │  │ Metrics │  │  Logs   │  │ Traces  │                         │    │
│  │  │ Library │  │ Library │  │ Library │                         │    │
│  │  └────┬────┘  └────┬────┘  └────┬────┘                         │    │
│  └───────┼────────────┼────────────┼────────────────────────────────┘    │
│          │            │            │                                     │
│          ▼            ▼            ▼                                     │
│  2. Collect and ship data                                               │
│                                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │  Prometheus  │  │   Fluentd/   │  │    Jaeger    │                  │
│  │  (scrapes    │  │   Logstash   │  │   Collector  │                  │
│  │   metrics)   │  │  (ships logs)│  │  (collects   │                  │
│  └──────┬───────┘  └──────┬───────┘  │   traces)    │                  │
│         │                 │          └──────┬───────┘                   │
│         │                 │                 │                           │
│         ▼                 ▼                 ▼                           │
│  3. Store data                                                          │
│                                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │  Prometheus  │  │Elasticsearch │  │    Jaeger    │                  │
│  │    TSDB      │  │   (logs)     │  │   Storage    │                  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                  │
│         │                 │                 │                           │
│         └─────────────────┼─────────────────┘                           │
│                           │                                             │
│                           ▼                                             │
│  4. Visualize and alert                                                 │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        GRAFANA                                   │    │
│  │  ┌──────────────────────────────────────────────────────────┐   │    │
│  │  │  Dashboard: Order Service                                 │   │    │
│  │  │  ┌────────────┐ ┌────────────┐ ┌────────────┐            │   │    │
│  │  │  │ Requests/s │ │ p99 Latency│ │ Error Rate │            │   │    │
│  │  │  │    1,234   │ │   145ms    │ │   0.02%    │            │   │    │
│  │  │  └────────────┘ └────────────┘ └────────────┘            │   │    │
│  │  └──────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                           │                                             │
│                           ▼                                             │
│  5. Alert on anomalies                                                  │
│                                                                          │
│  ┌──────────────┐                                                       │
│  │  Alertmanager│ ──► PagerDuty ──► On-call engineer                   │
│  │              │ ──► Slack                                             │
│  │              │ ──► Email                                             │
│  └──────────────┘                                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Real Systems at Real Companies

**Netflix**:

- Atlas: Custom time-series database for metrics
- Monitors millions of metrics per second
- Uses anomaly detection to find issues before users notice

**Google**:

- Borgmon (predecessor to Prometheus)
- Dapper (distributed tracing, inspired Jaeger/Zipkin)
- Every service has SLOs with automated alerting

**Amazon**:

- CloudWatch for metrics and logs
- X-Ray for distributed tracing
- Automated canary deployments with monitoring

### Common Monitoring Stacks

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    POPULAR MONITORING STACKS                             │
│                                                                          │
│  PROMETHEUS + GRAFANA (Open Source)                                     │
│  ──────────────────────────────────                                     │
│  Metrics: Prometheus                                                     │
│  Visualization: Grafana                                                  │
│  Alerting: Alertmanager                                                  │
│  Cost: Free (self-hosted)                                               │
│  Best for: Kubernetes, cloud-native                                      │
│                                                                          │
│  ELK STACK (Open Source)                                                │
│  ───────────────────────                                                │
│  Logs: Elasticsearch + Logstash + Kibana                                │
│  Or: Elasticsearch + Fluentd + Kibana (EFK)                             │
│  Cost: Free (self-hosted) or Elastic Cloud                              │
│  Best for: Log aggregation, search                                      │
│                                                                          │
│  DATADOG (SaaS)                                                          │
│  ─────────────                                                           │
│  All-in-one: Metrics, Logs, Traces, APM                                 │
│  Cost: $15-35/host/month                                                │
│  Best for: Teams wanting managed solution                               │
│                                                                          │
│  AWS CLOUDWATCH (Cloud Provider)                                        │
│  ───────────────────────────────                                        │
│  Metrics: CloudWatch Metrics                                            │
│  Logs: CloudWatch Logs                                                   │
│  Traces: X-Ray                                                           │
│  Cost: Pay per use                                                       │
│  Best for: AWS-native applications                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6️⃣ How to Implement Monitoring

### Spring Boot with Micrometer and Prometheus

```java
// MetricsConfiguration.java
package com.example.monitoring;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Component;

/**
 * Custom metrics for business operations.
 */
@Component
public class OrderMetrics {

    private final Counter ordersCreated;
    private final Counter ordersFailed;
    private final Timer orderProcessingTime;

    public OrderMetrics(MeterRegistry registry) {
        // Counter: Increments only, good for counting events
        this.ordersCreated = Counter.builder("orders.created")
            .description("Number of orders created")
            .tag("service", "order-service")
            .register(registry);

        this.ordersFailed = Counter.builder("orders.failed")
            .description("Number of failed orders")
            .tag("service", "order-service")
            .register(registry);

        // Timer: Measures duration and count
        this.orderProcessingTime = Timer.builder("orders.processing.time")
            .description("Time to process an order")
            .publishPercentiles(0.5, 0.9, 0.95, 0.99)  // p50, p90, p95, p99
            .register(registry);
    }

    public void recordOrderCreated() {
        ordersCreated.increment();
    }

    public void recordOrderFailed(String reason) {
        ordersFailed.increment();
    }

    public Timer.Sample startTimer() {
        return Timer.start();
    }

    public void stopTimer(Timer.Sample sample) {
        sample.stop(orderProcessingTime);
    }
}
```

### Structured Logging

```java
// LoggingConfiguration.java
package com.example.monitoring;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import org.springframework.stereotype.Service;

/**
 * Demonstrates structured logging best practices.
 */
@Service
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public Order createOrder(OrderRequest request) {
        // Add context to all logs in this request
        MDC.put("orderId", request.orderId());
        MDC.put("userId", request.userId());
        MDC.put("traceId", getTraceId());  // From distributed tracing

        try {
            log.info("Creating order: amount={}, items={}",
                request.amount(), request.items().size());

            // Process order...
            Order order = processOrder(request);

            log.info("Order created successfully: status={}", order.status());
            return order;

        } catch (PaymentException e) {
            // Structured error logging
            log.error("Payment failed: errorCode={}, message={}",
                e.getErrorCode(), e.getMessage(), e);
            throw e;

        } finally {
            // Clean up MDC
            MDC.clear();
        }
    }
}
```

### Logback Configuration for JSON Logs

```xml
<!-- logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- JSON format for production (machine-readable) -->
    <springProfile name="prod">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <includeMdcKeyName>orderId</includeMdcKeyName>
                <includeMdcKeyName>userId</includeMdcKeyName>
                <includeMdcKeyName>traceId</includeMdcKeyName>
            </encoder>
        </appender>
    </springProfile>

    <!-- Human-readable format for development -->
    <springProfile name="dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
    </springProfile>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>

</configuration>
```

### Health Check Implementation

```java
// CustomHealthIndicator.java
package com.example.monitoring;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

/**
 * Custom health indicator for payment service dependency.
 */
@Component
public class PaymentServiceHealthIndicator implements HealthIndicator {

    private final PaymentClient paymentClient;

    public PaymentServiceHealthIndicator(PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }

    @Override
    public Health health() {
        try {
            long start = System.currentTimeMillis();
            boolean healthy = paymentClient.healthCheck();
            long latency = System.currentTimeMillis() - start;

            if (healthy) {
                return Health.up()
                    .withDetail("latency_ms", latency)
                    .build();
            } else {
                return Health.down()
                    .withDetail("reason", "Health check returned false")
                    .build();
            }

        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### Application Configuration

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus, info
  endpoint:
    health:
      show-details: always
      show-components: always
      probes:
        enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: order-service
      environment: ${ENVIRONMENT:dev}
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.9, 0.95, 0.99
      slo:
        http.server.requests: 100ms, 200ms, 500ms, 1s

# Logging
logging:
  level:
    root: INFO
    com.example: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
```

### Prometheus Alert Rules

```yaml
# prometheus-alerts.yml
groups:
  - name: order-service
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          /
          sum(rate(http_server_requests_seconds_count[5m]))
          > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} (> 1%)"

      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, 
            sum(rate(http_server_requests_seconds_bucket[5m])) by (le)
          ) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High p99 latency"
          description: "p99 latency is {{ $value }}s (> 500ms)"

      # Service down
      - alert: ServiceDown
        expr: up{job="order-service"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Order service is down"
          description: "Order service instance {{ $labels.instance }} is down"

      # High CPU
      - alert: HighCPU
        expr: |
          process_cpu_usage{job="order-service"} > 0.8
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage"
          description: "CPU usage is {{ $value | humanizePercentage }}"
```

### Grafana Dashboard (JSON)

```json
{
  "title": "Order Service Dashboard",
  "panels": [
    {
      "title": "Request Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_server_requests_seconds_count{application=\"order-service\"}[1m]))",
          "legendFormat": "Requests/sec"
        }
      ]
    },
    {
      "title": "Response Time (p99)",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket{application=\"order-service\"}[5m])) by (le))",
          "legendFormat": "p99"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "singlestat",
      "targets": [
        {
          "expr": "sum(rate(http_server_requests_seconds_count{application=\"order-service\",status=~\"5..\"}[5m])) / sum(rate(http_server_requests_seconds_count{application=\"order-service\"}[5m])) * 100",
          "legendFormat": "Error %"
        }
      ]
    }
  ]
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

**1. Alert fatigue**

```
WRONG: Alert on every metric exceeding any threshold
       - 100 alerts/day
       - Team ignores all alerts
       - Real issues get missed

RIGHT: Alert only on actionable, urgent issues
       - 1-2 alerts/week
       - Each alert gets attention
       - Clear runbook for each alert
```

**2. Not correlating metrics, logs, and traces**

```
WRONG:
       Metrics show high latency
       Can't find which requests are slow
       Logs don't have request IDs

RIGHT:
       Metrics show high latency
       Click through to traces for slow requests
       Traces link to logs with same trace ID
       Full picture in minutes
```

**3. Logging too much or too little**

```
WRONG (too much):
       Log every function call
       Log full request/response bodies
       Result: TB of logs, can't find anything, high costs

WRONG (too little):
       Only log errors
       No context in error logs
       Result: Can't debug issues

RIGHT:
       Log business events (order created, payment processed)
       Log errors with context
       Use DEBUG level for detailed info (disabled in prod)
```

**4. No baseline**

```
WRONG: Alert when latency > 500ms
       But you don't know normal latency
       Alert might fire constantly or never

RIGHT:
       Establish baseline: Normal latency is 100-150ms
       Alert when latency > 3x baseline (450ms)
       Or use anomaly detection
```

---

## 8️⃣ When NOT to Over-Monitor

### Situations Where Less is More

1. **Early-stage startup**: Ship features first, add monitoring incrementally
2. **Simple applications**: Don't need Prometheus for a static website
3. **Development environments**: Basic logging is usually enough
4. **One-off scripts**: Not worth instrumenting

### Signs You're Over-Monitoring

- Dashboard has 50 panels, no one looks at them
- Hundreds of metrics, can't find the important ones
- Alerts fire constantly, team has alert fatigue
- More time maintaining monitoring than the application

---

## 9️⃣ Comparison: Monitoring Approaches

| Approach                    | Pros                 | Cons                 | Best For                  |
| --------------------------- | -------------------- | -------------------- | ------------------------- |
| Self-hosted (Prometheus)    | Free, flexible       | Operational overhead | Large teams, custom needs |
| SaaS (Datadog)              | Easy setup, features | Cost at scale        | Small-medium teams        |
| Cloud provider (CloudWatch) | Integrated, no setup | Vendor lock-in       | All-in on one cloud       |
| APM (New Relic)             | Deep insights        | Expensive            | Performance-critical apps |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What are the three pillars of observability?**

A: The three pillars are metrics, logs, and traces. Metrics are numerical measurements over time (like CPU usage, request count, latency). They answer "how much" and "how fast." Logs are textual records of events (like errors, user actions). They answer "what happened." Traces follow a request across multiple services. They answer "where did time go" in distributed systems. Together, they give complete visibility: metrics detect problems, traces locate them, logs explain them.

**Q: What should you monitor in a web application?**

A: I'd use the Four Golden Signals: (1) Latency: response time percentiles (p50, p95, p99), not just average. (2) Traffic: requests per second to understand load. (3) Errors: error rate (5xx errors, exceptions). (4) Saturation: resource usage (CPU, memory, connections). I'd also monitor business metrics like orders per minute, and dependency health like database response time. The goal is knowing if users are having a good experience and if the system is healthy.

### L5 (Mid-Level) Questions

**Q: How would you set up alerting that doesn't cause alert fatigue?**

A: Key principles: (1) Alert on symptoms, not causes. Alert on "high error rate" not "high CPU" (CPU might be fine). (2) Only alert on actionable issues. If no one can do anything at 3 AM, don't page. (3) Set appropriate thresholds with hysteresis. Alert when error rate > 5% for 5 minutes, not on every spike. (4) Have clear severity levels: P1 pages immediately, P2 during business hours, P3 creates a ticket. (5) Include runbooks: every alert should link to "what to do." (6) Review alerts regularly: if an alert never fires or always fires, fix it. (7) Use anomaly detection for dynamic thresholds instead of static ones.

**Q: How do you correlate logs, metrics, and traces?**

A: The key is consistent identifiers. Every request gets a trace ID generated at the edge (API gateway or first service). This trace ID is: (1) Added to all log messages via MDC (Mapped Diagnostic Context). (2) Propagated to downstream services in HTTP headers (e.g., X-Trace-Id). (3) Included in metrics as a tag for high-cardinality debugging. When investigating an issue: start with metrics dashboard showing the problem (high latency spike at 2 PM), click through to traces from that time period, find slow traces, click through to logs with that trace ID. Tools like Datadog, Grafana, and Jaeger support this correlation out of the box.

### L6 (Senior) Questions

**Q: How would you design a monitoring strategy for a microservices architecture?**

A: I'd implement monitoring at multiple levels: (1) Infrastructure: Node-level metrics (CPU, memory, disk, network) for capacity planning. (2) Platform: Kubernetes metrics (pod health, deployments, resource requests vs usage). (3) Service: Per-service golden signals (latency, traffic, errors, saturation). (4) Business: Domain metrics (orders/minute, conversion rate, revenue). For distributed tracing, I'd use OpenTelemetry for vendor-neutral instrumentation. Every service propagates trace context. For logs, structured JSON format with consistent fields (service, trace_id, user_id). Centralized log aggregation with retention policies. For alerting, SLO-based alerts (error budget consumption) rather than threshold alerts. Dashboards: one overview dashboard, then drill-down dashboards per service. I'd also implement synthetic monitoring (canary requests) to detect issues before users do.

**Q: How do you balance monitoring detail with cost and performance?**

A: It's about being strategic: (1) Metrics: Use histograms instead of individual percentiles (more efficient). Limit cardinality (don't use user_id as a metric tag). Aggregate at the source. (2) Logs: Log at appropriate levels (INFO in prod, DEBUG off). Sample high-volume logs (log 1% of successful requests, 100% of errors). Set retention policies (7 days hot, 30 days warm, archive). (3) Traces: Sample traces (1% of normal requests, 100% of errors and slow requests). Use head-based sampling for consistency. (4) Costs: Monitor your monitoring costs. Set up alerts for unexpected spikes in data ingestion. Use tiered storage. Consider self-hosted for high-volume metrics. The goal is enough data to debug issues without drowning in noise or costs.

---

## 1️⃣1️⃣ One Clean Mental Summary

Monitoring is your system's dashboard. The three pillars are metrics (numbers over time: CPU, latency, errors), logs (text records of events), and traces (request flow across services). Use the Four Golden Signals: latency, traffic, errors, and saturation. Good alerts are actionable, urgent, and rare. Correlate everything with trace IDs so you can go from "something is wrong" (metrics) to "here's where" (traces) to "here's why" (logs). The goal isn't collecting data, it's answering questions: Is the system healthy? Are users happy? What's broken and why?
