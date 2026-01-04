# 👁️ Observability

## 0️⃣ Prerequisites

Before diving into observability, you should understand:

- **Logging Basics**: What logs are and why applications write them
- **Metrics Concepts**: Numerical measurements over time (CPU usage, request count)
- **Distributed Systems**: Services communicating over networks (covered in Phase 1)
- **HTTP Basics**: Request/response model (covered in Phase 2)

Quick refresher on **logging**: Applications write log messages to record events, errors, and information. Logs are typically text files or streams that developers read to understand what happened.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Without Observability

Imagine debugging a production issue:

**Problem 1: The "Black Box" Application**

```
3:00 AM: User reports "payment failed"
3:05 AM: Check application logs
3:10 AM: Logs show "Error processing payment"
3:15 AM: But why? What payment? Which user? What step failed?

No context. No trace. Just an error message.
```

**Problem 2: The "Needle in Haystack" Problem**

```
Application generates 10 million log lines per day.
User reports issue from 2 hours ago.

To find the issue:
1. Search through 10 million lines
2. Filter by time range
3. Filter by user ID
4. Filter by error type
5. Still have 1000s of lines to read

Time to find issue: Hours
```

**Problem 3: The "Silent Failure"**

```
Service A calls Service B
Service B calls Service C
Service C fails silently

User sees: "Request timeout"
Developer sees: Service A logged timeout
But why? Which service failed? Where in the chain?

No visibility into the request flow.
```

**Problem 4: The "Metric Blindness"**

```
Application is slow.
But:
- Is it CPU? Memory? Network? Database?
- Is it all users or specific users?
- Is it all endpoints or specific endpoints?
- When did it start?

No metrics. No way to know.
```

**Problem 5: The "Reactive Debugging"**

```
Issue discovered when users complain.
By then:
- Issue has been happening for hours
- Many users affected
- Context is lost
- Hard to reproduce

No proactive monitoring. Always reactive.
```

### What Breaks Without Observability

| Scenario | Without Observability | With Observability |
|----------|---------------------|-------------------|
| Debugging | Hours of log searching | Trace shows exact path |
| Performance issues | "It's slow" | Metrics show CPU/memory/DB |
| User complaints | "Something broke" | Logs show exact error |
| Proactive detection | None | Alerts before users notice |
| Root cause analysis | Guesswork | Full request trace |

---

## 2️⃣ Intuition and Mental Model

### The Three Pillars of Observability

Think of observability as **three ways to understand your system**:

**1. Logs**: "What happened?"
- Discrete events
- "User logged in at 3:15 PM"
- "Payment failed: insufficient funds"
- Like a diary of events

**2. Metrics**: "How much? How often?"
- Aggregated measurements
- "100 requests per second"
- "CPU usage: 75%"
- Like a dashboard with gauges

**3. Traces**: "How did this request flow through the system?"
- Request journey across services
- "Request went: API → Auth → Payment → Database"
- Like a map of the request path

```
┌─────────────────────────────────────────────────────────────────┐
│                  THREE PILLARS OF OBSERVABILITY                  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    LOGS                                   │   │
│  │  "What happened?"                                         │   │
│  │                                                           │   │
│  │  2024-01-15 10:30:15 INFO  User alice logged in          │   │
│  │  2024-01-15 10:30:20 ERROR Payment failed: timeout        │   │
│  │  2024-01-15 10:30:25 INFO  Order created: order-123       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    METRICS                                │   │
│  │  "How much? How often?"                                   │   │
│  │                                                           │   │
│  │  Requests/sec: 1,250                                      │   │
│  │  Error rate: 0.5%                                         │   │
│  │  CPU usage: 65%                                           │   │
│  │  Response time p95: 250ms                                 │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    TRACES                                 │   │
│  │  "How did this flow?"                                     │   │
│  │                                                           │   │
│  │  Request ID: abc123                                       │   │
│  │    ├─ API Gateway (10ms)                                 │   │
│  │    ├─ Auth Service (50ms)                                │   │
│  │    ├─ Payment Service (200ms)                            │   │
│  │    │   └─ Database (180ms)                               │   │
│  │    └─ Notification Service (30ms)                         │   │
│  │  Total: 290ms                                             │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ Logging Deep Dive

### Structured Logging

**Unstructured logs** (hard to parse):
```
2024-01-15 10:30:15 User alice logged in from 192.168.1.1
2024-01-15 10:30:20 Payment failed for user bob amount 99.99
```

**Structured logs** (JSON, easy to parse):
```json
{
  "timestamp": "2024-01-15T10:30:15Z",
  "level": "INFO",
  "message": "User logged in",
  "userId": "alice",
  "ip": "192.168.1.1",
  "service": "auth-service"
}
```

### Log Levels

| Level | Use Case | Example |
|-------|----------|---------|
| TRACE | Very detailed debugging | "Entering function processPayment" |
| DEBUG | Debugging information | "Payment amount: 99.99" |
| INFO | Normal operations | "User logged in" |
| WARN | Warning, but continues | "Retry attempt 3 of 5" |
| ERROR | Error occurred | "Database connection failed" |
| FATAL | Critical error, app may crash | "Out of memory" |

### Java Logging with SLF4J and Logback

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>2.0.9</version>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.4.14</version>
</dependency>
```

```java
// Using SLF4J
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;

@RestController
public class PaymentController {
    
    private static final Logger logger = LoggerFactory.getLogger(PaymentController.class);
    
    @PostMapping("/payments")
    public PaymentResponse processPayment(@RequestBody PaymentRequest request) {
        // Set correlation ID for tracing
        String correlationId = UUID.randomUUID().toString();
        MDC.put("correlationId", correlationId);
        MDC.put("userId", request.getUserId());
        
        try {
            logger.info("Processing payment", 
                kv("amount", request.getAmount()),
                kv("currency", request.getCurrency()));
            
            PaymentResponse response = paymentService.process(request);
            
            logger.info("Payment processed successfully",
                kv("paymentId", response.getPaymentId()));
            
            return response;
        } catch (PaymentException e) {
            logger.error("Payment processing failed",
                kv("errorCode", e.getCode()),
                kv("errorMessage", e.getMessage()),
                e);
            throw e;
        } finally {
            MDC.clear();
        }
    }
}
```

### Logback Configuration

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp/>
                <version/>
                <logLevel/>
                <message/>
                <mdc/>
                <stackTrace/>
            </providers>
        </encoder>
    </appender>
    
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application-%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp/>
                <version/>
                <logLevel/>
                <message/>
                <mdc/>
                <stackTrace/>
            </providers>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="FILE" />
    </root>
    
    <logger name="com.example" level="DEBUG" />
</configuration>
```

### Correlation IDs

```java
// Filter to add correlation ID to all requests
@Component
public class CorrelationIdFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        // Get correlation ID from header or generate new one
        String correlationId = httpRequest.getHeader("X-Correlation-ID");
        if (correlationId == null) {
            correlationId = UUID.randomUUID().toString();
        }
        
        // Add to MDC (Mapped Diagnostic Context)
        MDC.put("correlationId", correlationId);
        
        try {
            // Add to response header
            ((HttpServletResponse) response).setHeader("X-Correlation-ID", correlationId);
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

**Benefits**: Trace a request across all services using correlation ID.

---

## 4️⃣ Metrics Deep Dive

### RED Metrics

**R**ate: Requests per second
**E**rrors: Error rate
**D**uration: Response time

```java
@Service
public class PaymentService {
    
    private final Counter requestCounter;
    private final Counter errorCounter;
    private final Timer responseTimer;
    
    public PaymentService(MeterRegistry meterRegistry) {
        this.requestCounter = Counter.builder("payment.requests")
            .description("Total payment requests")
            .tag("service", "payment")
            .register(meterRegistry);
        
        this.errorCounter = Counter.builder("payment.errors")
            .description("Payment errors")
            .tag("service", "payment")
            .register(meterRegistry);
        
        this.responseTimer = Timer.builder("payment.duration")
            .description("Payment processing duration")
            .tag("service", "payment")
            .register(meterRegistry);
    }
    
    public PaymentResponse process(PaymentRequest request) {
        return responseTimer.recordCallable(() -> {
            requestCounter.increment();
            
            try {
                PaymentResponse response = processInternal(request);
                return response;
            } catch (Exception e) {
                errorCounter.increment(
                    Tags.of("error_type", e.getClass().getSimpleName())
                );
                throw e;
            }
        });
    }
}
```

### USE Metrics

**U**tilization: Resource usage (CPU, memory, disk)
**S**aturation: Queue length, wait time
**E**rrors: Error count

```java
// System metrics (usually auto-collected)
// CPU utilization: system.cpu.usage
// Memory utilization: jvm.memory.used
// Disk utilization: disk.usage
// Thread saturation: jvm.threads.live
```

### Prometheus Integration

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>1.12.0</version>
</dependency>
```

```java
// Spring Boot auto-configures Prometheus
// Metrics available at /actuator/prometheus
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'payment-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['payment-service:8080']
```

### Grafana Dashboard

```json
{
  "dashboard": {
    "title": "Payment Service Metrics",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(payment_requests_total[5m])"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(payment_errors_total[5m])"
          }
        ]
      },
      {
        "title": "Response Time p95",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(payment_duration_seconds_bucket[5m]))"
          }
        ]
      }
    ]
  }
}
```

---

## 5️⃣ Distributed Tracing

### What is Distributed Tracing?

Tracing follows a request as it flows through multiple services.

```
Request: POST /api/orders
  │
  ├─ API Gateway (10ms)
  │   │
  │   ├─ Auth Service (50ms)
  │   │   └─ Database (45ms)
  │   │
  │   ├─ Order Service (200ms)
  │   │   ├─ Inventory Service (80ms)
  │   │   │   └─ Database (75ms)
  │   │   └─ Payment Service (100ms)
  │   │       └─ External API (95ms)
  │   │
  │   └─ Notification Service (30ms)
  │
Total: 290ms
```

### OpenTelemetry Integration

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
    <version>1.32.0</version>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-sdk</artifactId>
    <version>1.32.0</version>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
    <version>2.0.0</version>
</dependency>
```

```java
@Service
public class PaymentService {
    
    @Autowired
    private Tracer tracer;
    
    public PaymentResponse process(PaymentRequest request) {
        Span span = tracer.nextSpan()
            .name("process-payment")
            .tag("payment.amount", request.getAmount())
            .tag("payment.currency", request.getCurrency())
            .start();
        
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            // Business logic
            PaymentResponse response = processInternal(request);
            
            span.tag("payment.id", response.getPaymentId());
            span.tag("payment.status", "success");
            
            return response;
        } catch (Exception e) {
            span.tag("payment.status", "error");
            span.tag("error", true);
            span.event("error", Attributes.of(
                AttributeKey.stringKey("error.message"), e.getMessage()
            ));
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### Jaeger Integration

```yaml
# docker-compose.yml
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI
      - "14268:14268"  # HTTP collector
```

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% sampling (adjust for production)
```

---

## 6️⃣ ELK Stack (Elasticsearch, Logstash, Kibana)

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    ELK STACK ARCHITECTURE                        │
│                                                                  │
│  Applications                                                   │
│       │                                                         │
│       │ Logs                                                    │
│       ▼                                                         │
│  ┌──────────────┐                                               │
│  │  Logstash    │  Parses, transforms, enriches logs          │
│  └──────────────┘                                               │
│       │                                                         │
│       │ Indexed logs                                            │
│       ▼                                                         │
│  ┌──────────────┐                                               │
│  │ Elasticsearch│  Stores and indexes logs                    │
│  └──────────────┘                                               │
│       │                                                         │
│       │ Query                                                   │
│       ▼                                                         │
│  ┌──────────────┐                                               │
│  │   Kibana     │  Visualizes and searches logs               │
│  └──────────────┘                                               │
└─────────────────────────────────────────────────────────────────┘
```

### Logstash Configuration

```ruby
# logstash.conf
input {
  beats {
    port => 5044
  }
}

filter {
  json {
    source => "message"
  }
  
  date {
    match => [ "timestamp", "ISO8601" ]
  }
  
  mutate {
    add_field => { "environment" => "production" }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

### Kibana Queries

```
# Find all errors in last hour
level:ERROR AND @timestamp:[now-1h TO now]

# Find logs for specific user
userId:alice AND @timestamp:[now-1d TO now]

# Find slow requests
duration_ms:>1000 AND @timestamp:[now-1h TO now]
```

---

## 7️⃣ Alerting

### Alert Rules

```yaml
# Prometheus alert rules
groups:
  - name: payment_service
    rules:
      - alert: HighErrorRate
        expr: rate(payment_errors_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate in payment service"
          description: "Error rate is {{ $value }} errors/sec"
      
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(payment_duration_seconds_bucket[5m])) > 1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High latency in payment service"
          description: "p95 latency is {{ $value }}s"
```

### Alertmanager Configuration

```yaml
# alertmanager.yml
route:
  group_by: ['alertname', 'severity']
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
    - match:
        severity: warning
      receiver: 'slack'

receivers:
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'your-service-key'
  
  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/...'
        channel: '#alerts'
```

---

## 8️⃣ Best Practices

### Logging Best Practices

1. **Use structured logging** (JSON)
2. **Include correlation IDs** for tracing
3. **Log at appropriate levels** (don't log everything as ERROR)
4. **Don't log sensitive data** (passwords, credit cards)
5. **Use consistent format** across services
6. **Log context** (userId, requestId, etc.)

### Metrics Best Practices

1. **Use standard metrics** (RED, USE)
2. **Label appropriately** (but avoid high cardinality)
3. **Set up dashboards** for key metrics
4. **Define SLOs** and alert on violations
5. **Monitor business metrics** (not just technical)

### Tracing Best Practices

1. **Sample appropriately** (100% in dev, 1-10% in prod)
2. **Keep traces lightweight** (don't add too many spans)
3. **Use correlation IDs** to link logs and traces
4. **Set reasonable timeouts** for trace collection

---

## 9️⃣ Interview Follow-Up Questions

### Q1: "What's the difference between logging and metrics?"

**Answer**:
**Logs** are discrete events with detailed context. "User alice logged in at 3:15 PM from IP 192.168.1.1". High volume, detailed, used for debugging specific issues.

**Metrics** are aggregated measurements over time. "100 logins per minute, average response time 250ms". Lower volume, aggregated, used for monitoring trends and alerting.

Use logs when you need to understand what happened in detail. Use metrics when you need to understand trends and patterns.

### Q2: "How would you implement distributed tracing in a microservices architecture?"

**Answer**:
Key components:

1. **Correlation IDs**: Generate at entry point (API Gateway), propagate via headers through all services.

2. **Tracing library**: Use OpenTelemetry or similar. Instrument services to create spans.

3. **Trace context propagation**: Pass trace context (trace ID, span ID) via HTTP headers between services.

4. **Trace collection**: Send traces to collector (Jaeger, Zipkin).

5. **Sampling**: Sample traces (1-10% in production) to reduce overhead.

Implementation: Add tracing library to each service. Middleware/interceptor automatically creates spans for incoming requests and propagates context to outgoing requests.

### Q3: "What metrics would you monitor for a payment service?"

**Answer**:
Key metrics:

**RED metrics**:
- Rate: Payment requests per second
- Errors: Payment failure rate, by error type
- Duration: Payment processing time (p50, p95, p99)

**Business metrics**:
- Payment success rate
- Revenue per hour/day
- Average transaction amount
- Payment method distribution

**Infrastructure metrics**:
- CPU, memory, database connections
- External API latency (payment gateway)
- Queue depth (if using queues)

**Alert thresholds**:
- Error rate > 1%
- p95 latency > 2 seconds
- Payment success rate < 95%

### Q4: "How do you handle log volume at scale?"

**Answer**:
Strategies:

1. **Structured logging**: JSON format enables efficient filtering and parsing.

2. **Log levels**: Use appropriate levels. Don't log DEBUG in production.

3. **Sampling**: Sample verbose logs (log 1 in 100 DEBUG messages).

4. **Log aggregation**: Centralize logs (ELK, Splunk) for efficient search.

5. **Retention policies**: Delete old logs (keep 30-90 days).

6. **Indexing**: Index important fields (userId, error type) for fast search.

7. **Archival**: Move old logs to cold storage (S3) instead of deleting.

8. **Filtering**: Don't log sensitive or high-volume data unnecessarily.

### Q5: "What's the difference between monitoring and observability?"

**Answer**:
**Monitoring** is about watching known metrics and alerting when thresholds are exceeded. You know what to look for. "CPU usage > 80%", "Error rate > 1%".

**Observability** is about understanding system behavior through logs, metrics, and traces. You can answer questions you didn't know to ask. "Why is this user's request slow?" - you explore traces, logs, and metrics to find the answer.

Monitoring answers "Is something wrong?" Observability answers "Why is something wrong?" and "What's happening?"

---

## 🔟 One Clean Mental Summary

Observability provides visibility into system behavior through three pillars: logs (what happened), metrics (how much/how often), and traces (how requests flow). Structured logging (JSON) enables efficient parsing. Metrics (RED, USE) track system health. Distributed tracing follows requests across services using correlation IDs.

Key tools: ELK stack for log aggregation, Prometheus + Grafana for metrics, Jaeger/Zipkin for tracing. Alerting notifies when thresholds are exceeded.

The key insight: Observability lets you understand system behavior, not just detect problems. With good observability, you can answer "why" questions, not just "what" questions.




