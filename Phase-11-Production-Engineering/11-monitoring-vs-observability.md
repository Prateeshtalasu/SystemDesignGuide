# 📊 Monitoring vs Observability

## 0️⃣ Prerequisites

Before diving into monitoring vs observability, you should understand:

- **Observability Basics**: The three pillars (logs, metrics, traces) from Topic 10
- **Metrics Concepts**: What metrics are and how they're collected
- **Alerting Basics**: How alerts work when thresholds are exceeded

Quick refresher on **monitoring**: Monitoring involves watching system metrics and alerting when values exceed predefined thresholds. It answers "Is something wrong?"

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Limitations of Traditional Monitoring

**Problem 1: The "Known Unknowns"**

```
Monitoring setup:
- Alert if CPU > 80%
- Alert if error rate > 1%
- Alert if memory > 90%

Issue occurs:
- CPU: 60% (no alert)
- Error rate: 0.5% (no alert)
- Memory: 70% (no alert)

But users report: "Application is slow"

Monitoring didn't catch it because we didn't know to monitor the right thing.
```

**Problem 2: The "Alert Fatigue"**

```
100 alerts configured:
- CPU > 80% (fires 50 times/day)
- Memory > 90% (fires 30 times/day)
- Disk > 85% (fires 20 times/day)
- Network latency > 100ms (fires 40 times/day)

Total: 140 alerts/day
Most are false positives or not actionable.

Engineers start ignoring alerts.
Real issues get missed.
```

**Problem 3: The "Why?" Question**

```
Alert fires: "Error rate > 1%"
Engineer checks: Yes, error rate is 1.2%

But:
- Which errors?
- Which users?
- Which endpoints?
- What's the root cause?

Monitoring tells you something is wrong.
It doesn't tell you why.
```

**Problem 4: The "Unknown Unknowns"**

```
New issue appears:
- Never seen before
- No alert configured
- No metric tracked
- No way to detect it

Monitoring only catches what you already know to look for.
```

### What Breaks Without Observability

| Scenario | Monitoring Only | With Observability |
|----------|----------------|-------------------|
| Unknown issues | Missed | Discovered through exploration |
| Root cause | "Something is wrong" | "Database query slow on user X" |
| Debugging | Hours of investigation | Minutes with traces |
| Proactive detection | Limited to known patterns | Can discover new patterns |
| User experience | "System is slow" | "User Y's request took 5s because..." |

---

## 2️⃣ Intuition and Mental Model

### The Car Dashboard Analogy

**Monitoring** is like a car's warning lights:
- Check engine light (alert)
- Low fuel light (alert)
- Oil pressure light (alert)

You know something is wrong, but not why.

**Observability** is like having:
- Warning lights (monitoring)
- Plus: OBD-II scanner (detailed diagnostics)
- Plus: Trip computer (historical data)
- Plus: GPS tracking (request traces)

You can diagnose the exact problem.

### Monitoring vs Observability

```
┌─────────────────────────────────────────────────────────────────┐
│                    MONITORING                                    │
│                                                                  │
│  Predefined metrics + thresholds                                │
│                                                                  │
│  CPU > 80% → Alert                                             │
│  Error rate > 1% → Alert                                        │
│  Memory > 90% → Alert                                           │
│                                                                  │
│  Answers: "Is something wrong?"                                 │
│  Limitations: Only catches known issues                         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  OBSERVABILITY                                   │
│                                                                  │
│  Logs + Metrics + Traces                                        │
│                                                                  │
│  Explore and understand system behavior                         │
│                                                                  │
│  "Why is user X's request slow?"                                │
│  → Check trace: Request went through 5 services                │
│  → Check logs: Service C had database timeout                  │
│  → Check metrics: Database connection pool exhausted            │
│                                                                  │
│  Answers: "Why is something wrong?" and "What's happening?"     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ Monitoring: What It Is

### Monitoring Characteristics

**1. Predefined Metrics**
```yaml
# Prometheus alert rules
- alert: HighCPU
  expr: cpu_usage > 80
  for: 5m

- alert: HighErrorRate
  expr: error_rate > 0.01
  for: 5m
```

**2. Threshold-Based Alerts**
- Set thresholds
- Alert when exceeded
- Reactive (alerts after issue occurs)

**3. Dashboards**
- Pre-built dashboards
- Known metrics
- Historical trends

**4. Known Patterns**
- Monitor what you know can go wrong
- Based on past incidents
- Limited to known failure modes

### Monitoring Maturity Levels

**Level 1: No Monitoring**
- No visibility
- Issues discovered by users

**Level 2: Basic Monitoring**
- CPU, memory, disk
- Uptime monitoring
- Basic alerts

**Level 3: Application Monitoring**
- Application metrics (requests, errors)
- Business metrics (revenue, conversions)
- Alerting on key metrics

**Level 4: Advanced Monitoring**
- Custom metrics
- Predictive alerts
- Anomaly detection

---

## 4️⃣ Observability: What It Is

### Observability Characteristics

**1. Exploratory**
- Ask questions you didn't know to ask
- Discover new patterns
- Investigate unknown issues

**2. Rich Context**
- Logs with full context
- Traces showing request flow
- Metrics with dimensions

**3. Ad-Hoc Queries**
- Query logs: "Show me all errors for user X"
- Query traces: "Show me slow requests"
- Query metrics: "Show me error rate by endpoint"

**4. Unknown Unknowns**
- Discover issues you didn't know existed
- Understand system behavior
- Learn from data

### Observability Maturity Model

**Level 1: Basic Logging**
- Unstructured logs
- No correlation
- Hard to search

**Level 2: Structured Logging**
- JSON logs
- Correlation IDs
- Centralized logging

**Level 3: Metrics + Logs**
- Application metrics
- Structured logs
- Basic dashboards

**Level 4: Full Observability**
- Logs + Metrics + Traces
- Distributed tracing
- Rich context
- Can answer any question

---

## 5️⃣ SLI, SLO, SLA

### Definitions

**SLI (Service Level Indicator)**: What you measure
- "Request success rate"
- "Response time p95"
- "Availability percentage"

**SLO (Service Level Objective)**: Target for SLI
- "99.9% of requests succeed"
- "p95 response time < 500ms"
- "99.95% uptime"

**SLA (Service Level Agreement)**: Contract with users
- "If SLO not met, refund 10%"
- Legal commitment

### Example

```
Service: Payment API

SLI: Request success rate
Measurement: (successful_requests / total_requests) * 100

SLO: 99.9% success rate
Target: >= 99.9%

SLA: If success rate < 99.9% for 1 hour, refund affected customers
```

### Error Budgets

```
SLO: 99.9% uptime
Means: 0.1% downtime allowed

In a month (30 days = 720 hours):
Error budget: 720 * 0.001 = 0.72 hours = 43.2 minutes

If you use 20 minutes this month:
Remaining budget: 23.2 minutes

If you use all budget:
- Stop deploying new features
- Focus on stability
- Rebuild budget over time
```

### Implementing SLOs

```java
@Service
public class SLOService {
    
    private final MeterRegistry meterRegistry;
    private final Counter successCounter;
    private final Counter totalCounter;
    
    public SLOService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.successCounter = Counter.builder("requests.success")
            .register(meterRegistry);
        this.totalCounter = Counter.builder("requests.total")
            .register(meterRegistry);
    }
    
    public void recordRequest(boolean success) {
        totalCounter.increment();
        if (success) {
            successCounter.increment();
        }
        
        // Calculate current SLI
        double successRate = successCounter.count() / totalCounter.count();
        
        // Alert if below SLO (99.9%)
        if (successRate < 0.999) {
            // Alert: SLO violation
        }
    }
}
```

---

## 6️⃣ Cardinality Management

### What is Cardinality?

**Cardinality**: Number of unique combinations of label values

```prometheus
# Low cardinality (good)
http_requests_total{method="GET", status="200"}  # 10 combinations

# High cardinality (bad)
http_requests_total{method="GET", status="200", user_id="user-123"}  # Millions of combinations
```

### Why Cardinality Matters

High cardinality metrics:
- Consume massive storage
- Slow down queries
- Can crash Prometheus

```
Example:
Metric with user_id label
1 million users
= 1 million time series
= Gigabytes of data
= Slow queries
= Prometheus OOM
```

### Best Practices

**1. Avoid High-Cardinality Labels**
```java
// BAD: user_id as label
Counter.builder("requests")
    .tag("user_id", userId)  // High cardinality!
    .register(registry);

// GOOD: Aggregate instead
Counter.builder("requests")
    .tag("user_type", getUserType(userId))  // Low cardinality
    .register(registry);
```

**2. Use Logs for High-Cardinality Data**
```java
// High-cardinality data → Logs
logger.info("Request processed", 
    kv("user_id", userId),  // OK in logs
    kv("request_id", requestId));

// Low-cardinality data → Metrics
counter.increment("requests.total",
    Tags.of("method", method, "status", status));
```

**3. Sample High-Cardinality Metrics**
```java
// Sample 1% of requests for detailed metrics
if (Math.random() < 0.01) {
    detailedMetrics.record(userId, endpoint);
}
```

---

## 7️⃣ Monitoring Best Practices

### 1. Monitor the Right Things

**RED Metrics** (for services):
- Rate: Requests per second
- Errors: Error rate
- Duration: Response time

**USE Metrics** (for infrastructure):
- Utilization: CPU, memory, disk
- Saturation: Queue length
- Errors: Error count

**Business Metrics**:
- Revenue per hour
- Conversion rate
- Active users

### 2. Set Meaningful Alerts

```yaml
# GOOD: Actionable alert
- alert: PaymentErrorRateHigh
  expr: rate(payment_errors_total[5m]) > 0.01
  for: 5m
  annotations:
    summary: "Payment error rate is {{ $value }}"
    runbook: "https://wiki/runbooks/payment-errors"

# BAD: Noise alert
- alert: CPUSlightlyHigh
  expr: cpu_usage > 50  # Too low threshold, fires constantly
```

### 3. Use Runbooks

```markdown
# Runbook: Payment Error Rate High

## Symptoms
- Error rate > 1%
- Alert: PaymentErrorRateHigh

## Investigation Steps
1. Check error logs: `level:ERROR service:payment`
2. Check error types: `payment_errors_total by (error_type)`
3. Check external dependencies: `payment_gateway_latency`
4. Check recent deployments

## Common Causes
- Payment gateway down
- Database connection issues
- Invalid payment data

## Resolution
1. If payment gateway: Check gateway status page
2. If database: Check connection pool metrics
3. If invalid data: Check recent code changes

## Escalation
- If not resolved in 15 minutes: Escalate to payment team
```

### 4. Dashboard Design

**Good Dashboard**:
- Shows key metrics at a glance
- Organized by service/component
- Time range selector
- Drill-down capability

**Bad Dashboard**:
- Too many metrics (information overload)
- No context
- Static (doesn't update)
- No drill-down

---

## 8️⃣ Observability Best Practices

### 1. Rich Context in Logs

```java
// GOOD: Rich context
logger.info("Payment processed",
    kv("payment_id", paymentId),
    kv("user_id", userId),
    kv("amount", amount),
    kv("currency", currency),
    kv("payment_method", method),
    kv("duration_ms", duration));

// BAD: Minimal context
logger.info("Payment processed");
```

### 2. Correlation IDs

```java
// Add correlation ID to all logs and traces
String correlationId = UUID.randomUUID().toString();
MDC.put("correlationId", correlationId);

// Now you can trace a request across all services
// Search: correlationId:abc-123
```

### 3. Structured Logging

```json
{
  "timestamp": "2024-01-15T10:30:15Z",
  "level": "INFO",
  "message": "Payment processed",
  "correlationId": "abc-123",
  "userId": "user-456",
  "paymentId": "pay-789",
  "duration_ms": 250
}
```

### 4. Distributed Tracing

```java
// Automatically trace requests across services
@RestController
public class PaymentController {
    
    @Autowired
    private Tracer tracer;
    
    @PostMapping("/payments")
    public PaymentResponse process(@RequestBody PaymentRequest request) {
        Span span = tracer.nextSpan()
            .name("process-payment")
            .start();
        
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            return paymentService.process(request);
        } finally {
            span.end();
        }
    }
}
```

---

## 9️⃣ Interview Follow-Up Questions

### Q1: "What's the difference between monitoring and observability?"

**Answer**:
**Monitoring** is about watching predefined metrics and alerting when thresholds are exceeded. It answers "Is something wrong?" but only for things you already know to monitor.

**Observability** is about understanding system behavior through logs, metrics, and traces. It lets you explore and answer questions you didn't know to ask. It answers "Why is something wrong?" and "What's happening?"

Think of it this way: Monitoring is like a car's warning lights (you know something is wrong). Observability is like having a diagnostic scanner (you can find out exactly what's wrong).

### Q2: "How do you define SLIs and SLOs for a service?"

**Answer**:
Process:

1. **Identify user-facing metrics**: What matters to users? (Availability, latency, correctness)

2. **Define SLIs**: Choose measurable indicators
   - Availability: Uptime percentage
   - Latency: p95 response time
   - Correctness: Error rate

3. **Set SLO targets**: Based on user needs and business requirements
   - Availability: 99.9% (43 minutes downtime/month)
   - Latency: p95 < 500ms
   - Correctness: Error rate < 0.1%

4. **Calculate error budgets**: SLO target - 100% = allowed failure
   - 99.9% uptime = 0.1% downtime allowed

5. **Monitor and alert**: Alert when approaching error budget limits

6. **Review and adjust**: Regularly review SLOs based on actual performance and user needs

### Q3: "How do you handle high-cardinality metrics?"

**Answer**:
Strategies:

1. **Avoid high-cardinality labels**: Don't use user_id, request_id as metric labels. Use in logs instead.

2. **Aggregate**: Group by categories (user_type instead of user_id).

3. **Sample**: Sample high-cardinality metrics (1% of requests).

4. **Use logs for detail**: High-cardinality data belongs in logs, not metrics.

5. **Limit label combinations**: Keep total time series under control (thousands, not millions).

Example: Instead of `requests_total{user_id="123"}`, use `requests_total{user_type="premium"}` and log user_id in structured logs.

### Q4: "What metrics would you monitor for a microservices architecture?"

**Answer**:
Key metrics by layer:

**Service level (RED)**:
- Request rate per service
- Error rate per service
- Latency (p50, p95, p99) per service

**Infrastructure level (USE)**:
- CPU, memory, disk per service
- Network I/O
- Database connections

**Business level**:
- Revenue, conversions
- User activity
- Feature usage

**Dependencies**:
- External API latency
- Database query performance
- Cache hit rates
- Message queue depth

**Cross-cutting**:
- Distributed trace sampling
- Service mesh metrics (if using)
- API Gateway metrics

I'd also set up SLOs per service and monitor error budgets.

### Q5: "How do you balance monitoring and observability?"

**Answer**:
They're complementary, not alternatives:

**Monitoring** for:
- Known issues (alert on these)
- SLO tracking
- Dashboards for common metrics
- Automated responses

**Observability** for:
- Debugging unknown issues
- Understanding system behavior
- Ad-hoc investigations
- Learning and improvement

Best practice: Use monitoring to detect issues (alerts), then use observability to understand them (logs, traces, metrics exploration). Monitor the critical path, observe everything.

Example: Alert on high error rate (monitoring), then use traces and logs to find which service and why (observability).

---

## 🔟 One Clean Mental Summary

Monitoring watches predefined metrics and alerts when thresholds are exceeded. It answers "Is something wrong?" but only for known issues. Observability provides logs, metrics, and traces to explore system behavior. It answers "Why is something wrong?" and enables discovering unknown issues.

SLIs measure what matters (success rate, latency). SLOs set targets (99.9% uptime). Error budgets track remaining failure allowance. High-cardinality metrics (user_id labels) should be avoided; use logs for detail instead.

The key insight: Monitoring detects problems, observability explains them. Use both: monitor critical metrics for alerts, observe everything for debugging and understanding.






