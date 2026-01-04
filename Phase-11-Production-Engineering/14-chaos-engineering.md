# 🔥 Chaos Engineering

## 0️⃣ Prerequisites

Before diving into chaos engineering, you should understand:

- **Distributed Systems**: Services communicating over networks (Phase 5)
- **Resilience Patterns**: Circuit breakers, retries, timeouts (Phase 7)
- **Observability**: Logs, metrics, traces (Topic 10)
- **Incident Response**: How to handle production issues (Topic 12)

Quick refresher on **resilience**: Resilience is the ability of a system to handle failures gracefully. Resilient systems degrade gracefully rather than failing completely when components fail.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Before Chaos Engineering

**Problem 1: The "It Works in Staging" Fallacy**

```
Staging environment:
- 2 servers
- 1 database
- 100 test users
- No real traffic

Production environment:
- 50 servers
- 10 databases (sharded)
- 10 million users
- 100,000 requests/second

Feature works in staging.
Feature breaks in production.
Why? Different scale, different failure modes.
```

**Problem 2: The "We Think We're Resilient" Problem**

```
Team says: "We have circuit breakers!"
Team says: "We have retries!"
Team says: "We have fallbacks!"

Reality:
- Circuit breaker configured wrong
- Retries cause thundering herd
- Fallback never tested

First real failure: Everything breaks.
```

**Problem 3: The "Unknown Dependencies"**

```
Service A depends on:
- Service B (known)
- Service C (known)
- Service D (unknown, hidden in library)
- External API (unknown, called by Service C)

Service D goes down.
Service A fails.
Team: "We didn't know we depended on Service D!"
```

**Problem 4: The "Cascading Failure"**

```
Database slows down (not down, just slow).
Service A retries, holds connections.
Service A connection pool exhausted.
Service B calls Service A, times out.
Service B retries, holds connections.
Service B connection pool exhausted.
Service C calls Service B...

Small issue cascades to full outage.
```

**Problem 5: The "Recovery Failure"**

```
Database fails over to replica.
Application reconnects.
But: Connection pool doesn't refresh.
Application still trying to connect to old primary.

Failover worked. Application didn't.
```

### What Breaks Without Chaos Engineering

| Scenario | Without Chaos Engineering | With Chaos Engineering |
|----------|--------------------------|----------------------|
| Failure discovery | In production, during incident | In controlled experiments |
| Confidence | "We think it's resilient" | "We know it handles X failure" |
| Dependencies | Unknown until they fail | Mapped and tested |
| Recovery | Untested | Verified |
| Team readiness | Panic during incidents | Practiced and prepared |

---

## 2️⃣ Intuition and Mental Model

### The Fire Drill Analogy

Think of chaos engineering as **fire drills for your systems**.

**Without fire drills**:
- No one knows exit routes
- Fire extinguishers might be empty
- Alarms might not work
- Panic during real fire

**With fire drills**:
- Everyone knows what to do
- Equipment tested regularly
- Problems found and fixed
- Calm during real fire

### Chaos Engineering Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHAOS ENGINEERING CYCLE                       │
│                                                                  │
│  1. STEADY STATE HYPOTHESIS                                     │
│     Define what "normal" looks like                             │
│     "System processes 1000 requests/sec with <1% error rate"    │
│                                                                  │
│  2. INTRODUCE FAILURE                                           │
│     Inject controlled failure                                   │
│     "Kill 1 of 3 database replicas"                            │
│                                                                  │
│  3. OBSERVE                                                     │
│     Monitor system behavior                                     │
│     "Does error rate stay <1%?"                                 │
│                                                                  │
│  4. LEARN                                                       │
│     Analyze results                                             │
│     "System handled failure" or "System failed"                 │
│                                                                  │
│  5. IMPROVE                                                     │
│     Fix weaknesses found                                        │
│     "Add connection pool refresh on failover"                   │
│                                                                  │
│  6. REPEAT                                                      │
│     Continuously test resilience                                │
└─────────────────────────────────────────────────────────────────┘
```

### Principles of Chaos Engineering

**1. Build a Hypothesis Around Steady State**
- Define normal behavior (metrics, SLOs)
- Hypothesis: "System maintains steady state despite failure"

**2. Vary Real-World Events**
- Simulate real failures (server crash, network partition)
- Not just theoretical failures

**3. Run Experiments in Production**
- Production has unique characteristics
- Staging doesn't catch all issues

**4. Automate Experiments to Run Continuously**
- One-time tests aren't enough
- Systems change, new failures emerge

**5. Minimize Blast Radius**
- Start small
- Have kill switch
- Limit impact

---

## 3️⃣ How It Works Internally

### Chaos Experiment Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHAOS EXPERIMENT                              │
│                                                                  │
│  Experiment: Database Failover                                  │
│                                                                  │
│  1. SCOPE                                                       │
│     Target: payment-db-primary                                  │
│     Blast radius: 10% of traffic                                │
│     Duration: 5 minutes                                         │
│                                                                  │
│  2. STEADY STATE                                                │
│     Metric: payment_success_rate                                │
│     Expected: >= 99.9%                                          │
│     Metric: payment_latency_p95                                 │
│     Expected: < 500ms                                           │
│                                                                  │
│  3. HYPOTHESIS                                                  │
│     "When primary database fails, system fails over to         │
│      replica and maintains steady state within 30 seconds"     │
│                                                                  │
│  4. EXPERIMENT                                                  │
│     Action: Kill primary database pod                          │
│     Method: kubectl delete pod payment-db-primary              │
│                                                                  │
│  5. ROLLBACK                                                    │
│     Automatic: Kubernetes restarts pod                         │
│     Manual: kubectl scale deployment payment-db --replicas=3   │
│                                                                  │
│  6. ABORT CONDITIONS                                            │
│     If error_rate > 5%: Abort immediately                      │
│     If latency_p95 > 2s: Abort immediately                     │
└─────────────────────────────────────────────────────────────────┘
```

### Failure Injection Types

**1. Infrastructure Failures**
```
- Server crash (kill process, terminate instance)
- Disk full (fill disk with data)
- CPU stress (consume CPU cycles)
- Memory pressure (allocate memory)
- Network partition (block traffic)
```

**2. Application Failures**
```
- Exception injection (throw errors)
- Latency injection (add delays)
- Response corruption (return bad data)
- Dependency failure (mock failed responses)
```

**3. External Dependency Failures**
```
- Third-party API down
- DNS failure
- Certificate expiration
- Rate limiting
```

---

## 4️⃣ Simulation: Chaos Engineering in Practice

### Step 1: Simple Failure Injection with Spring Boot

```java
// Chaos configuration
@Configuration
@ConditionalOnProperty(name = "chaos.enabled", havingValue = "true")
public class ChaosConfig {
    
    @Value("${chaos.latency.enabled:false}")
    private boolean latencyEnabled;
    
    @Value("${chaos.latency.ms:0}")
    private int latencyMs;
    
    @Value("${chaos.error.enabled:false}")
    private boolean errorEnabled;
    
    @Value("${chaos.error.rate:0.0}")
    private double errorRate;
    
    @Bean
    public ChaosInterceptor chaosInterceptor() {
        return new ChaosInterceptor(latencyEnabled, latencyMs, errorEnabled, errorRate);
    }
}
```

```java
// Chaos interceptor
@Component
public class ChaosInterceptor implements HandlerInterceptor {
    
    private final boolean latencyEnabled;
    private final int latencyMs;
    private final boolean errorEnabled;
    private final double errorRate;
    private final Random random = new Random();
    
    public ChaosInterceptor(boolean latencyEnabled, int latencyMs, 
                           boolean errorEnabled, double errorRate) {
        this.latencyEnabled = latencyEnabled;
        this.latencyMs = latencyMs;
        this.errorEnabled = errorEnabled;
        this.errorRate = errorRate;
    }
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) throws Exception {
        
        // Inject latency
        if (latencyEnabled && latencyMs > 0) {
            Thread.sleep(latencyMs);
        }
        
        // Inject errors
        if (errorEnabled && random.nextDouble() < errorRate) {
            throw new ChaosException("Chaos monkey strikes!");
        }
        
        return true;
    }
}
```

```yaml
# application.yml - Enable chaos in non-prod
chaos:
  enabled: ${CHAOS_ENABLED:false}
  latency:
    enabled: ${CHAOS_LATENCY_ENABLED:false}
    ms: ${CHAOS_LATENCY_MS:0}
  error:
    enabled: ${CHAOS_ERROR_ENABLED:false}
    rate: ${CHAOS_ERROR_RATE:0.0}
```

### Step 2: Chaos Mesh for Kubernetes

```yaml
# chaos-mesh-pod-kill.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-payment
  namespace: chaos-testing
spec:
  action: pod-kill
  mode: one  # Kill one pod
  selector:
    namespaces:
      - payment
    labelSelectors:
      app: payment-service
  scheduler:
    cron: "@every 1h"  # Run every hour
```

```yaml
# chaos-mesh-network-delay.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-payment
  namespace: chaos-testing
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - payment
    labelSelectors:
      app: payment-service
  delay:
    latency: "100ms"
    correlation: "25"
    jitter: "50ms"
  duration: "5m"
```

```yaml
# chaos-mesh-cpu-stress.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cpu-stress-payment
  namespace: chaos-testing
spec:
  mode: one
  selector:
    namespaces:
      - payment
    labelSelectors:
      app: payment-service
  stressors:
    cpu:
      workers: 2
      load: 80  # 80% CPU load
  duration: "5m"
```

### Step 3: Litmus Chaos Experiments

```yaml
# litmus-pod-delete.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: payment-chaos
  namespace: payment
spec:
  appinfo:
    appns: payment
    applabel: app=payment-service
    appkind: deployment
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "30"
            - name: CHAOS_INTERVAL
              value: "10"
            - name: FORCE
              value: "false"
```

### Step 4: Automated Chaos Experiments in CI/CD

```yaml
# .github/workflows/chaos-test.yml
name: Chaos Testing

on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM
  workflow_dispatch:

jobs:
  chaos-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name chaos-cluster
      
      - name: Capture baseline metrics
        run: |
          ./scripts/capture-metrics.sh > baseline.json
      
      - name: Run chaos experiment
        run: |
          kubectl apply -f chaos/pod-kill-experiment.yaml
          sleep 300  # Wait 5 minutes
      
      - name: Capture experiment metrics
        run: |
          ./scripts/capture-metrics.sh > experiment.json
      
      - name: Validate steady state
        run: |
          ./scripts/validate-steady-state.sh baseline.json experiment.json
      
      - name: Cleanup
        if: always()
        run: |
          kubectl delete -f chaos/pod-kill-experiment.yaml
      
      - name: Report results
        run: |
          ./scripts/report-chaos-results.sh
```

---

## 5️⃣ Game Days

### What is a Game Day?

A **game day** is a planned chaos engineering exercise where the team practices responding to simulated failures.

```
┌─────────────────────────────────────────────────────────────────┐
│                    GAME DAY STRUCTURE                            │
│                                                                  │
│  BEFORE (1-2 weeks)                                             │
│  - Define scenarios                                             │
│  - Notify stakeholders                                          │
│  - Prepare rollback plans                                       │
│  - Set up monitoring                                            │
│                                                                  │
│  DURING (2-4 hours)                                             │
│  - Brief participants                                           │
│  - Run scenarios                                                │
│  - Observe and document                                         │
│  - Debrief after each scenario                                  │
│                                                                  │
│  AFTER (1 week)                                                 │
│  - Full debrief                                                 │
│  - Document findings                                            │
│  - Create action items                                          │
│  - Update runbooks                                              │
└─────────────────────────────────────────────────────────────────┘
```

### Game Day Scenario Template

```markdown
# Game Day Scenario: Database Failover

## Objective
Verify that the payment service handles database failover gracefully.

## Participants
- Facilitator: Alice
- Observers: Bob, Charlie
- On-call: Dave (simulated)

## Pre-conditions
- Payment service running with 3 replicas
- Database primary and 2 replicas running
- Monitoring dashboards open
- Incident channel created

## Scenario Steps
1. [T+0] Facilitator kills database primary pod
2. [T+0] Observe automatic failover to replica
3. [T+5m] Verify payment service reconnects
4. [T+10m] Run test transactions
5. [T+15m] Verify metrics return to normal

## Expected Outcome
- Failover completes within 30 seconds
- Error rate stays below 1%
- No manual intervention required

## Abort Criteria
- Error rate exceeds 5%
- Failover takes longer than 2 minutes
- Data corruption detected

## Rollback Plan
1. Manually promote replica to primary
2. Restart payment service pods
3. Restore from backup if needed
```

### Game Day Debrief Template

```markdown
# Game Day Debrief: Database Failover

## Date: 2024-01-15
## Participants: Alice, Bob, Charlie, Dave

## Scenario Results

### Scenario 1: Database Primary Failure
- **Outcome**: PASS
- **Failover time**: 25 seconds
- **Error rate during failover**: 0.3%
- **Notes**: Faster than expected

### Scenario 2: Network Partition
- **Outcome**: FAIL
- **Issue**: Circuit breaker didn't trigger
- **Error rate**: 15%
- **Notes**: Circuit breaker timeout too high

## Findings

### What Worked Well
1. Database failover was fast
2. Alerting triggered correctly
3. Runbook was accurate

### What Didn't Work
1. Circuit breaker configuration wrong
2. Connection pool didn't refresh
3. Logs were missing context

## Action Items
1. [ ] Fix circuit breaker timeout (Owner: Bob, Due: 2024-01-20)
2. [ ] Add connection pool refresh on failover (Owner: Charlie, Due: 2024-01-22)
3. [ ] Improve log context (Owner: Dave, Due: 2024-01-18)

## Runbook Updates
- Update database failover runbook with new recovery steps
- Add circuit breaker verification step
```

---

## 6️⃣ Chaos Engineering Tools

### Tool Comparison

| Tool | Type | Best For | Pros | Cons |
|------|------|----------|------|------|
| Chaos Monkey | Netflix OSS | Random instance termination | Simple, proven | Limited failure types |
| Chaos Mesh | Kubernetes | K8s chaos experiments | Rich features, CRD-based | K8s only |
| Litmus | Kubernetes | K8s chaos + observability | Good UI, experiments library | Complex setup |
| Gremlin | SaaS | Enterprise chaos | Easy to use, support | Expensive |
| Toxiproxy | Network | Network failures | Simple, lightweight | Network only |

### Chaos Monkey for Spring Boot

```xml
<!-- pom.xml -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>chaos-monkey-spring-boot</artifactId>
    <version>3.1.0</version>
</dependency>
```

```yaml
# application.yml
chaos:
  monkey:
    enabled: true
    watcher:
      controller: true
      restController: true
      service: true
      repository: true
    assaults:
      level: 5  # 1 in 5 requests affected
      latencyActive: true
      latencyRangeStart: 1000
      latencyRangeEnd: 3000
      exceptionsActive: true
      killApplicationActive: false
```

```java
// Enable via actuator
// POST /actuator/chaosmonkey/enable
// POST /actuator/chaosmonkey/assaults
// {
//   "level": 3,
//   "latencyActive": true,
//   "latencyRangeStart": 500,
//   "latencyRangeEnd": 1000
// }
```

### Toxiproxy for Network Chaos

```java
// Toxiproxy client
public class ToxiproxyConfig {
    
    private final ToxiproxyClient client;
    
    public ToxiproxyConfig() {
        this.client = new ToxiproxyClient("localhost", 8474);
    }
    
    public void addLatency(String proxyName, int latencyMs) throws IOException {
        Proxy proxy = client.getProxy(proxyName);
        proxy.toxics().latency("latency_downstream", ToxicDirection.DOWNSTREAM, latencyMs);
    }
    
    public void addTimeout(String proxyName, int timeoutMs) throws IOException {
        Proxy proxy = client.getProxy(proxyName);
        proxy.toxics().timeout("timeout_downstream", ToxicDirection.DOWNSTREAM, timeoutMs);
    }
    
    public void addBandwidth(String proxyName, long rate) throws IOException {
        Proxy proxy = client.getProxy(proxyName);
        proxy.toxics().bandwidth("bandwidth_downstream", ToxicDirection.DOWNSTREAM, rate);
    }
}
```

```yaml
# docker-compose.yml with Toxiproxy
services:
  toxiproxy:
    image: ghcr.io/shopify/toxiproxy:latest
    ports:
      - "8474:8474"  # API
      - "5433:5433"  # PostgreSQL proxy
      - "6380:6380"  # Redis proxy
    
  app:
    environment:
      - DATABASE_URL=jdbc:postgresql://toxiproxy:5433/mydb
      - REDIS_URL=redis://toxiproxy:6380
```

---

## 7️⃣ Tradeoffs and Common Mistakes

### Common Mistakes

**1. Running Chaos in Production Without Preparation**

```
BAD: "Let's just kill a pod and see what happens"

GOOD:
1. Define steady state hypothesis
2. Set up monitoring
3. Have rollback plan
4. Start with small blast radius
5. Run experiment
```

**2. No Abort Conditions**

```
BAD: Run experiment until it's "done"

GOOD:
- Abort if error_rate > 5%
- Abort if latency_p95 > 2s
- Abort if revenue drops > 1%
- Have kill switch ready
```

**3. Not Learning from Experiments**

```
BAD: "Experiment failed. Oh well."

GOOD:
1. Document what happened
2. Identify root cause
3. Create action items
4. Fix issues
5. Re-run experiment
```

**4. Only Testing Happy Path**

```
BAD: Only test "server crash" scenario

GOOD: Test variety of failures:
- Slow responses (latency)
- Partial failures (some requests fail)
- Cascading failures
- Recovery scenarios
```

**5. Chaos Without Observability**

```
BAD: Inject chaos, can't see what's happening

GOOD:
- Dashboards showing key metrics
- Alerts configured
- Distributed tracing enabled
- Logs with correlation IDs
```

---

## 8️⃣ Building Resilience

### Resilience Patterns to Test

**1. Circuit Breaker**
```java
@CircuitBreaker(name = "paymentGateway", fallbackMethod = "fallback")
public PaymentResponse processPayment(PaymentRequest request) {
    return paymentGateway.process(request);
}

public PaymentResponse fallback(PaymentRequest request, Exception e) {
    return PaymentResponse.pending("Payment queued for retry");
}
```

**Chaos test**: Inject failures to verify circuit breaker opens.

**2. Retry with Backoff**
```java
@Retry(name = "paymentGateway", fallbackMethod = "fallback")
public PaymentResponse processPayment(PaymentRequest request) {
    return paymentGateway.process(request);
}
```

**Chaos test**: Inject intermittent failures to verify retries work.

**3. Bulkhead**
```java
@Bulkhead(name = "paymentGateway", type = Bulkhead.Type.THREADPOOL)
public PaymentResponse processPayment(PaymentRequest request) {
    return paymentGateway.process(request);
}
```

**Chaos test**: Inject latency to verify bulkhead limits concurrent calls.

**4. Timeout**
```java
@TimeLimiter(name = "paymentGateway")
public CompletableFuture<PaymentResponse> processPayment(PaymentRequest request) {
    return CompletableFuture.supplyAsync(() -> paymentGateway.process(request));
}
```

**Chaos test**: Inject latency to verify timeout triggers.

---

## 9️⃣ Interview Follow-Up Questions

### Q1: "What is chaos engineering and why is it important?"

**Answer**:
Chaos engineering is the practice of intentionally injecting failures into a system to test its resilience. It's based on the principle that failures will happen in production, so it's better to discover weaknesses in controlled experiments than during real incidents.

Key benefits:
1. **Discover weaknesses**: Find issues before they cause outages
2. **Verify resilience**: Confirm that circuit breakers, retries, and fallbacks work
3. **Build confidence**: Know how system behaves under failure
4. **Train teams**: Practice incident response
5. **Improve systems**: Fix issues found during experiments

Example: Netflix's Chaos Monkey randomly terminates instances in production. This ensures their services can handle instance failures gracefully.

### Q2: "How do you design a chaos experiment?"

**Answer**:
Steps to design a chaos experiment:

1. **Define steady state**: What does "normal" look like? (e.g., 99.9% success rate, <500ms latency)

2. **Form hypothesis**: "System maintains steady state when X fails"

3. **Choose failure type**: Server crash, network partition, latency, etc.

4. **Set blast radius**: Start small (1 pod, 10% traffic). Expand if safe.

5. **Define abort conditions**: When to stop (error rate >5%, latency >2s)

6. **Plan rollback**: How to restore if things go wrong

7. **Run experiment**: Inject failure, observe behavior

8. **Analyze results**: Did system maintain steady state?

9. **Document and improve**: Fix weaknesses, update runbooks

Key principle: Start small, have kill switch, never run without monitoring.

### Q3: "What's the difference between chaos engineering and testing?"

**Answer**:
Key differences:

**Traditional Testing**:
- Tests known scenarios
- Pass/fail based on expected behavior
- Run in test environment
- Deterministic (same input → same output)
- Tests specific code paths

**Chaos Engineering**:
- Explores unknown failures
- Validates system-wide resilience
- Ideally run in production
- Non-deterministic (real-world conditions)
- Tests system behavior under stress

Analogy: Testing is like checking if a car's brakes work. Chaos engineering is like driving the car in various conditions (rain, snow, hills) to see how it handles.

They're complementary: Testing verifies correctness, chaos engineering verifies resilience.

### Q4: "How do you run chaos experiments safely in production?"

**Answer**:
Safety measures:

1. **Start small**: Begin with small blast radius (1 pod, 5% traffic)

2. **Monitor closely**: Have dashboards open, alerts configured

3. **Have kill switch**: Ability to stop experiment immediately

4. **Define abort conditions**: Automatic stop if metrics exceed thresholds

5. **Run during low traffic**: Start during off-peak hours

6. **Notify stakeholders**: Let support team know experiment is running

7. **Have rollback plan**: Know how to restore quickly

8. **Limit duration**: Short experiments (5-15 minutes)

9. **Gradual expansion**: Only increase blast radius after successful runs

10. **Document everything**: Record what happened for learning

Key insight: Production chaos is valuable because it tests real conditions, but safety is paramount.

### Q5: "What failures should you test with chaos engineering?"

**Answer**:
Categories of failures to test:

**Infrastructure**:
- Server/pod crash
- Disk full
- CPU exhaustion
- Memory pressure
- Network partition

**Application**:
- Service unavailable
- Slow responses (latency injection)
- Error responses
- Dependency failures

**External**:
- Third-party API down
- DNS failure
- Certificate expiration

**Data**:
- Database failover
- Cache eviction
- Data corruption (read-only)

**Recovery**:
- Failover to backup
- Auto-scaling triggers
- Circuit breaker recovery

Start with most likely failures (server crash, network issues), then expand to less common but high-impact scenarios.

---

## 🔟 One Clean Mental Summary

Chaos engineering is the practice of intentionally injecting failures to test system resilience. The process: define steady state (normal behavior), form hypothesis (system handles failure X), inject failure, observe behavior, learn and improve.

Key principles: build hypothesis around steady state, vary real-world events, run in production when safe, automate experiments, minimize blast radius. Tools include Chaos Mesh, Litmus, and Chaos Monkey for Spring Boot.

The key insight: Failures will happen. It's better to discover weaknesses in controlled experiments than during real incidents. Chaos engineering builds confidence that your system can handle failures gracefully.

