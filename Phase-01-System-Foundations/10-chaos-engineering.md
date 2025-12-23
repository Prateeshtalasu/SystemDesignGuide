# 🔥 Chaos Engineering: Building Confidence Through Controlled Failure

---

## 0️⃣ Prerequisites

Before understanding chaos engineering, you need to know:

- **Distributed System**: Multiple computers working together (covered in Topic 1).
- **Failure Modes**: How systems can fail (covered in Topic 7).
- **Monitoring**: How to observe system behavior (covered in Topic 9).
- **Resilience Patterns**: Circuit breakers, retries, fallbacks (covered in Topic 7).

If you understand that systems fail and we need to handle those failures gracefully, you're ready.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Point

You've built a distributed system with:

- Redundant servers
- Circuit breakers
- Retry logic
- Fallback mechanisms

But how do you know they actually work?

The only way to find out is when a real failure happens in production. By then, it's too late. Users are affected. Revenue is lost.

### What Systems Looked Like Before

Before chaos engineering:

- Hope that failover works
- Find out about weaknesses during real outages
- Untested disaster recovery plans
- "It should work" instead of "we know it works"

### What Breaks Without It

1. **Untested assumptions**: "The circuit breaker will trip" (but does it?)
2. **Hidden dependencies**: Didn't know Service A depends on Service B
3. **Configuration drift**: Failover worked 6 months ago, but not anymore
4. **Stale runbooks**: Documentation doesn't match reality
5. **Surprise failures**: First time seeing a failure mode is during an incident

### Real Examples of the Problem

**Amazon (2017)**: S3 outage took down thousands of websites. Many companies discovered their "redundant" systems all depended on S3.

**Knight Capital (2012)**: Deployment failure caused $440M loss. They had never tested what happens when old and new code run simultaneously.

**Facebook (2021)**: Configuration change caused global outage. Internal tools also depended on the same infrastructure, preventing engineers from fixing it.

---

## 2️⃣ Intuition and Mental Model

### The Fire Drill Analogy

Think of chaos engineering like fire drills:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE FIRE DRILL ANALOGY                                │
│                                                                          │
│  WITHOUT FIRE DRILLS:                                                    │
│  ─────────────────────                                                   │
│  • Fire alarms installed ✓                                              │
│  • Sprinklers installed ✓                                               │
│  • Exit signs posted ✓                                                  │
│  • Fire extinguishers placed ✓                                          │
│                                                                          │
│  Real fire happens:                                                      │
│  • Alarm doesn't work (battery dead)                                    │
│  • Exit blocked by furniture                                            │
│  • No one knows where to go                                             │
│  • Fire extinguisher expired                                            │
│  • CHAOS AND PANIC                                                       │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  WITH FIRE DRILLS:                                                       │
│  ─────────────────                                                       │
│  Regular drills reveal:                                                  │
│  • Alarm battery needs replacing                                        │
│  • Exit path needs clearing                                             │
│  • People need training on evacuation                                   │
│  • Extinguishers need inspection                                        │
│                                                                          │
│  Real fire happens:                                                      │
│  • Everyone knows what to do                                            │
│  • Equipment works as expected                                          │
│  • Calm, orderly evacuation                                             │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  CHAOS ENGINEERING = FIRE DRILLS FOR YOUR SOFTWARE                      │
│                                                                          │
│  Intentionally inject failures in controlled conditions                 │
│  to discover weaknesses before real incidents expose them.              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key insight**: The best time to discover your system's weaknesses is when you're prepared, not during a real crisis.

---

## 3️⃣ How It Works Internally

### Chaos Engineering Principles

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CHAOS ENGINEERING PRINCIPLES                          │
│                    (From Netflix's Chaos Engineering Book)               │
│                                                                          │
│  1. BUILD A HYPOTHESIS AROUND STEADY STATE BEHAVIOR                     │
│     ──────────────────────────────────────────────                      │
│     Define what "normal" looks like:                                    │
│     • Request success rate > 99.9%                                      │
│     • p99 latency < 200ms                                               │
│     • Orders processed per minute: 1000 ± 100                           │
│                                                                          │
│     Hypothesis: "When X fails, the system will remain in steady state"  │
│                                                                          │
│  2. VARY REAL-WORLD EVENTS                                              │
│     ────────────────────────                                            │
│     Inject failures that could really happen:                           │
│     • Server crashes                                                     │
│     • Network latency spikes                                            │
│     • Disk fills up                                                      │
│     • Third-party API becomes slow                                      │
│     • Region becomes unavailable                                        │
│                                                                          │
│  3. RUN EXPERIMENTS IN PRODUCTION                                       │
│     ─────────────────────────────                                       │
│     Test environments don't have:                                       │
│     • Real traffic patterns                                             │
│     • Real data volumes                                                  │
│     • Real infrastructure complexity                                    │
│     Production is the only true test                                    │
│                                                                          │
│  4. AUTOMATE EXPERIMENTS TO RUN CONTINUOUSLY                            │
│     ──────────────────────────────────────────                          │
│     • Don't just run once and forget                                    │
│     • Systems change, new weaknesses appear                             │
│     • Continuous chaos builds continuous confidence                     │
│                                                                          │
│  5. MINIMIZE BLAST RADIUS                                               │
│     ─────────────────────                                               │
│     • Start small (one instance, one region)                            │
│     • Have kill switches to stop experiments                            │
│     • Monitor closely during experiments                                │
│     • Be ready to roll back                                             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Types of Chaos Experiments

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CHAOS EXPERIMENT TYPES                                │
│                                                                          │
│  INFRASTRUCTURE CHAOS                                                    │
│  ─────────────────────                                                   │
│  • Kill instances/containers                                            │
│  • Terminate entire availability zones                                  │
│  • Fill up disk space                                                    │
│  • Exhaust memory                                                        │
│  • CPU stress                                                            │
│                                                                          │
│  NETWORK CHAOS                                                           │
│  ─────────────                                                           │
│  • Add latency to network calls                                         │
│  • Drop packets (packet loss)                                           │
│  • Partition network (split brain)                                      │
│  • DNS failures                                                          │
│  • Bandwidth throttling                                                  │
│                                                                          │
│  APPLICATION CHAOS                                                       │
│  ─────────────────                                                       │
│  • Inject exceptions                                                     │
│  • Slow down responses                                                   │
│  • Return error codes                                                    │
│  • Corrupt data                                                          │
│  • Kill processes                                                        │
│                                                                          │
│  DEPENDENCY CHAOS                                                        │
│  ────────────────                                                        │
│  • Make database slow                                                    │
│  • Make cache unavailable                                                │
│  • Third-party API errors                                               │
│  • Message queue delays                                                  │
│                                                                          │
│  HUMAN CHAOS (GameDay)                                                   │
│  ─────────────────────                                                   │
│  • Simulate on-call scenarios                                           │
│  • Test incident response                                               │
│  • Verify runbooks work                                                  │
│  • Practice communication                                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### The Chaos Experiment Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CHAOS EXPERIMENT LIFECYCLE                            │
│                                                                          │
│  1. DEFINE STEADY STATE                                                  │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │ Metrics that indicate "system is healthy":                   │     │
│     │ • Error rate < 0.1%                                          │     │
│     │ • p99 latency < 200ms                                        │     │
│     │ • Throughput: 1000 ± 50 RPS                                  │     │
│     └─────────────────────────────────────────────────────────────┘     │
│                              │                                           │
│                              ▼                                           │
│  2. FORM HYPOTHESIS                                                      │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │ "If we kill 1 of 3 database replicas,                        │     │
│     │  the system will failover to remaining replicas              │     │
│     │  and maintain steady state within 30 seconds"                │     │
│     └─────────────────────────────────────────────────────────────┘     │
│                              │                                           │
│                              ▼                                           │
│  3. DESIGN EXPERIMENT                                                    │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │ • Target: Database replica in us-east-1a                     │     │
│     │ • Action: Terminate instance                                 │     │
│     │ • Duration: Until failover complete or 5 minutes            │     │
│     │ • Abort conditions: Error rate > 5%, latency > 2s           │     │
│     │ • Rollback: Restart instance, verify health                 │     │
│     └─────────────────────────────────────────────────────────────┘     │
│                              │                                           │
│                              ▼                                           │
│  4. RUN EXPERIMENT                                                       │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │ • Notify team (or run during GameDay)                        │     │
│     │ • Start monitoring dashboards                                │     │
│     │ • Inject failure                                             │     │
│     │ • Observe system behavior                                    │     │
│     │ • Record all observations                                    │     │
│     └─────────────────────────────────────────────────────────────┘     │
│                              │                                           │
│                              ▼                                           │
│  5. ANALYZE RESULTS                                                      │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │ Hypothesis confirmed:                                        │     │
│     │ ✓ Failover completed in 15 seconds                          │     │
│     │ ✓ Error rate peaked at 0.5% during failover                 │     │
│     │ ✓ Latency spike to 500ms, recovered to normal               │     │
│     │                                                              │     │
│     │ OR Hypothesis disproved:                                     │     │
│     │ ✗ Failover took 3 minutes (expected 30 seconds)             │     │
│     │ ✗ Error rate hit 10% (expected < 1%)                        │     │
│     │ → Create ticket to fix failover configuration               │     │
│     └─────────────────────────────────────────────────────────────┘     │
│                              │                                           │
│                              ▼                                           │
│  6. IMPROVE AND REPEAT                                                   │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │ • Fix discovered weaknesses                                  │     │
│     │ • Update runbooks with learnings                            │     │
│     │ • Schedule next experiment                                   │     │
│     │ • Increase blast radius gradually                           │     │
│     └─────────────────────────────────────────────────────────────┘     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Netflix Chaos Monkey

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NETFLIX CHAOS MONKEY                                  │
│                                                                          │
│  What it does:                                                           │
│  Randomly terminates virtual machine instances in production            │
│                                                                          │
│  Why:                                                                    │
│  "The best way to avoid failure is to fail constantly"                  │
│  - Netflix                                                               │
│                                                                          │
│  Schedule:                                                               │
│  Runs during business hours (when engineers are available)              │
│  Doesn't run on weekends or holidays                                    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │    9 AM                                              5 PM        │    │
│  │      │                                                │          │    │
│  │      ├──────── Chaos Monkey Active ──────────────────┤          │    │
│  │      │                                                │          │    │
│  │      │    🐵 Kill    🐵 Kill    🐵 Kill               │          │    │
│  │      │    Instance   Instance   Instance              │          │    │
│  │      │                                                │          │    │
│  │      │    Services auto-recover, no user impact       │          │    │
│  │      │                                                │          │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  THE SIMIAN ARMY (Netflix's chaos tools):                               │
│  ────────────────────────────────────────                               │
│  • Chaos Monkey: Kills instances                                        │
│  • Chaos Kong: Kills entire regions                                     │
│  • Latency Monkey: Adds artificial delays                               │
│  • Conformity Monkey: Finds non-conforming instances                    │
│  • Janitor Monkey: Cleans up unused resources                           │
│  • Security Monkey: Finds security violations                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Simulation-First Explanation

### Running Your First Chaos Experiment

**Scenario**: Test that your order service handles database failures gracefully.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    EXPERIMENT: DATABASE FAILURE                          │
│                                                                          │
│  SETUP:                                                                  │
│  ───────                                                                 │
│  Order Service (3 instances) ──► PostgreSQL (Primary + 2 Replicas)     │
│                                                                          │
│  Circuit breaker configured:                                            │
│  • Failure threshold: 50%                                               │
│  • Timeout: 5 seconds                                                   │
│  • Fallback: Return cached order or error message                       │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STEADY STATE (before experiment):                                      │
│  ─────────────────────────────────                                      │
│  • Error rate: 0.02%                                                    │
│  • p99 latency: 150ms                                                   │
│  • Orders/minute: 500                                                   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  HYPOTHESIS:                                                             │
│  ───────────                                                             │
│  "When the database primary fails, the circuit breaker will open,       │
│   the system will return fallback responses, and recover within         │
│   60 seconds when database fails over to replica"                       │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  EXPERIMENT EXECUTION:                                                   │
│  ─────────────────────                                                   │
│                                                                          │
│  T+0s:   Start experiment                                               │
│          Inject: Block network to database primary                      │
│                                                                          │
│  T+5s:   First timeouts observed                                        │
│          Circuit breaker failure count: 5/10                            │
│          Error rate: 2%                                                 │
│                                                                          │
│  T+10s:  Circuit breaker OPENS                                          │
│          Fallback responses returned                                    │
│          Error rate: 0.5% (fallback failures)                           │
│          Latency: 50ms (fast fallback)                                  │
│                                                                          │
│  T+30s:  Database failover complete                                     │
│          Primary: Replica promoted                                       │
│          Application reconnecting...                                    │
│                                                                          │
│  T+45s:  Circuit breaker HALF-OPEN                                      │
│          Testing with limited requests                                  │
│                                                                          │
│  T+50s:  Circuit breaker CLOSED                                         │
│          Normal operation resumed                                       │
│          Error rate: 0.02%                                              │
│          Latency: 160ms                                                 │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  RESULTS:                                                                │
│  ────────                                                                │
│  ✓ Circuit breaker opened as expected                                  │
│  ✓ Fallback responses worked                                           │
│  ✓ Recovery within 60 seconds                                          │
│  ⚠️ 2% error spike during initial failure (before circuit opened)      │
│                                                                          │
│  ACTION ITEMS:                                                           │
│  ─────────────                                                           │
│  • Reduce circuit breaker timeout from 5s to 2s                        │
│  • Add retry for first failure before counting toward circuit          │
│  • Update runbook with observed failover time                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Real Systems at Real Companies

**Netflix**:

- Runs Chaos Monkey continuously in production
- Chaos Kong tests entire region failures
- Every service must survive instance termination
- "If you can't survive Chaos Monkey, you can't deploy"

**Amazon**:

- GameDay exercises simulate major failures
- Tests region evacuations regularly
- Teams must demonstrate resilience before launch

**Google**:

- DiRT (Disaster Recovery Testing)
- Annual large-scale failure simulations
- Tests everything from datacenter loss to key person unavailability

**Slack**:

- Disasterpiece Theater: Planned chaos exercises
- Tests failure scenarios with the whole company watching
- Builds confidence and trains incident response

### Chaos Engineering Tools

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CHAOS ENGINEERING TOOLS                               │
│                                                                          │
│  CHAOS MONKEY (Netflix)                                                  │
│  ─────────────────────                                                   │
│  • Randomly terminates instances                                        │
│  • AWS focused                                                           │
│  • Part of Simian Army                                                   │
│  • Open source                                                           │
│                                                                          │
│  GREMLIN (Commercial)                                                    │
│  ─────────────────────                                                   │
│  • Full chaos platform                                                   │
│  • Infrastructure, network, application chaos                           │
│  • SaaS with enterprise features                                        │
│  • Great UI and reporting                                               │
│                                                                          │
│  LITMUS CHAOS (CNCF)                                                    │
│  ───────────────────                                                    │
│  • Kubernetes-native                                                     │
│  • ChaosHub with pre-built experiments                                  │
│  • GitOps friendly                                                       │
│  • Open source                                                           │
│                                                                          │
│  CHAOS MESH (CNCF)                                                       │
│  ─────────────────                                                       │
│  • Kubernetes-native                                                     │
│  • Pod, network, I/O, time chaos                                        │
│  • Dashboard included                                                    │
│  • Open source                                                           │
│                                                                          │
│  AWS FAULT INJECTION SIMULATOR                                          │
│  ─────────────────────────────                                          │
│  • AWS managed service                                                   │
│  • EC2, ECS, EKS, RDS experiments                                       │
│  • Integrated with AWS services                                         │
│  • Pay per use                                                           │
│                                                                          │
│  TOXIPROXY (Shopify)                                                    │
│  ───────────────────                                                    │
│  • Network chaos proxy                                                   │
│  • Add latency, timeouts, bandwidth limits                              │
│  • Great for testing                                                     │
│  • Open source                                                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6️⃣ How to Implement Chaos Engineering

### Simple Chaos with Spring Boot

```java
// ChaosController.java
package com.example.chaos;

import org.springframework.web.bind.annotation.*;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Simple chaos injection endpoints for testing.
 *
 * WARNING: Only enable in non-production or during controlled experiments!
 */
@RestController
@RequestMapping("/chaos")
@ConditionalOnProperty(name = "chaos.enabled", havingValue = "true")
public class ChaosController {

    private final AtomicBoolean failureEnabled = new AtomicBoolean(false);
    private final AtomicInteger latencyMs = new AtomicInteger(0);
    private final AtomicInteger failureRate = new AtomicInteger(0);

    /**
     * Enable/disable complete failure mode.
     * All requests will fail with 500 error.
     */
    @PostMapping("/failure")
    public String toggleFailure(@RequestParam boolean enabled) {
        failureEnabled.set(enabled);
        return "Failure mode: " + (enabled ? "ENABLED" : "DISABLED");
    }

    /**
     * Add artificial latency to all requests.
     */
    @PostMapping("/latency")
    public String setLatency(@RequestParam int ms) {
        latencyMs.set(ms);
        return "Latency set to: " + ms + "ms";
    }

    /**
     * Set percentage of requests that should fail.
     */
    @PostMapping("/failure-rate")
    public String setFailureRate(@RequestParam int percent) {
        failureRate.set(Math.min(100, Math.max(0, percent)));
        return "Failure rate set to: " + percent + "%";
    }

    /**
     * Reset all chaos settings.
     */
    @PostMapping("/reset")
    public String reset() {
        failureEnabled.set(false);
        latencyMs.set(0);
        failureRate.set(0);
        return "Chaos settings reset";
    }

    /**
     * Get current chaos settings.
     */
    @GetMapping("/status")
    public ChaosStatus getStatus() {
        return new ChaosStatus(
            failureEnabled.get(),
            latencyMs.get(),
            failureRate.get()
        );
    }

    // Getters for use in interceptor
    public boolean isFailureEnabled() { return failureEnabled.get(); }
    public int getLatencyMs() { return latencyMs.get(); }
    public int getFailureRate() { return failureRate.get(); }

    public record ChaosStatus(boolean failureEnabled, int latencyMs, int failureRate) {}
}
```

```java
// ChaosInterceptor.java
package com.example.chaos;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import java.util.Random;

/**
 * Intercepts requests and applies chaos based on settings.
 */
@Component
@ConditionalOnProperty(name = "chaos.enabled", havingValue = "true")
public class ChaosInterceptor implements HandlerInterceptor {

    private final ChaosController chaosController;
    private final Random random = new Random();

    public ChaosInterceptor(ChaosController chaosController) {
        this.chaosController = chaosController;
    }

    @Override
    public boolean preHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler) throws Exception {

        // Skip chaos endpoints themselves
        if (request.getRequestURI().startsWith("/chaos")) {
            return true;
        }

        // Apply failure mode
        if (chaosController.isFailureEnabled()) {
            response.sendError(500, "Chaos: Failure mode enabled");
            return false;
        }

        // Apply random failure rate
        int failureRate = chaosController.getFailureRate();
        if (failureRate > 0 && random.nextInt(100) < failureRate) {
            response.sendError(500, "Chaos: Random failure");
            return false;
        }

        // Apply latency
        int latency = chaosController.getLatencyMs();
        if (latency > 0) {
            Thread.sleep(latency);
        }

        return true;
    }
}
```

### Using Chaos Mesh (Kubernetes)

```yaml
# chaos-mesh-pod-kill.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-experiment
  namespace: order-service
spec:
  action: pod-kill
  mode: one # Kill one pod
  selector:
    namespaces:
      - order-service
    labelSelectors:
      app: order-service
  scheduler:
    cron: "0 10 * * 1-5" # Every weekday at 10 AM
---
# chaos-mesh-network-delay.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-experiment
  namespace: order-service
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - order-service
    labelSelectors:
      app: order-service
  delay:
    latency: "200ms"
    jitter: "50ms"
    correlation: "50"
  duration: "5m"
  scheduler:
    cron: "30 14 * * 1-5" # Every weekday at 2:30 PM
```

### Using Toxiproxy for Network Chaos

```java
// ToxiproxyTest.java
package com.example.chaos;

import eu.rekawek.toxiproxy.Proxy;
import eu.rekawek.toxiproxy.ToxiproxyClient;
import eu.rekawek.toxiproxy.model.ToxicDirection;
import org.junit.jupiter.api.*;

/**
 * Integration tests using Toxiproxy to simulate network issues.
 */
public class DatabaseResilienceTest {

    private static ToxiproxyClient toxiproxy;
    private static Proxy databaseProxy;

    @BeforeAll
    static void setup() throws Exception {
        // Connect to Toxiproxy (running in Docker or locally)
        toxiproxy = new ToxiproxyClient("localhost", 8474);

        // Create proxy for database
        // App connects to localhost:15432, proxy forwards to real DB at localhost:5432
        databaseProxy = toxiproxy.createProxy(
            "database",
            "localhost:15432",
            "localhost:5432"
        );
    }

    @AfterEach
    void resetProxy() throws Exception {
        // Remove all toxics after each test
        databaseProxy.toxics().getAll().forEach(toxic -> {
            try { toxic.remove(); } catch (Exception e) {}
        });
    }

    @Test
    void shouldHandleDatabaseLatency() throws Exception {
        // Add 500ms latency to database connections
        databaseProxy.toxics()
            .latency("db-latency", ToxicDirection.DOWNSTREAM, 500);

        // Test that application handles slow database gracefully
        // Circuit breaker should eventually open
        // Fallback should be returned
    }

    @Test
    void shouldHandleDatabaseTimeout() throws Exception {
        // Add timeout (connection hangs)
        databaseProxy.toxics()
            .timeout("db-timeout", ToxicDirection.DOWNSTREAM, 10000);

        // Test that application times out and fails gracefully
    }

    @Test
    void shouldHandleConnectionReset() throws Exception {
        // Reset connections after 1000 bytes
        databaseProxy.toxics()
            .resetPeer("db-reset", ToxicDirection.DOWNSTREAM, 1000);

        // Test that application handles connection resets
    }

    @Test
    void shouldHandlePacketLoss() throws Exception {
        // Drop 10% of packets
        databaseProxy.toxics()
            .bandwidth("db-bandwidth", ToxicDirection.DOWNSTREAM, 0)
            .setToxicity(0.1f);  // 10% of time

        // Test that application handles packet loss
    }
}
```

### GameDay Runbook Template

```markdown
# GameDay: Database Failover Exercise

## Overview

- **Date**: 2024-12-23
- **Duration**: 2 hours (10 AM - 12 PM)
- **Participants**: Platform Team, Order Service Team, On-call
- **Objective**: Verify database failover works correctly

## Pre-requisites

- [ ] Notify stakeholders 24 hours in advance
- [ ] Ensure monitoring dashboards are ready
- [ ] Verify rollback procedures
- [ ] Confirm on-call engineer availability
- [ ] Test communication channels (Slack, PagerDuty)

## Steady State Definition

- Error rate: < 0.1%
- p99 latency: < 200ms
- Orders per minute: 500 ± 50

## Experiment Plan

### Experiment 1: Kill Database Replica (10:15 AM)

**Hypothesis**: System continues operating normally when one replica is killed.

**Steps**:

1. Record baseline metrics
2. Kill replica in us-east-1b
3. Observe metrics for 5 minutes
4. Verify replica rejoins cluster

**Expected Outcome**: No user impact, replica auto-recovers.

**Abort Conditions**: Error rate > 1% or latency > 1s

### Experiment 2: Kill Database Primary (10:45 AM)

**Hypothesis**: Automatic failover to replica within 30 seconds.

**Steps**:

1. Record baseline metrics
2. Kill primary database instance
3. Observe failover behavior
4. Verify application reconnects to new primary

**Expected Outcome**:

- Failover completes in < 30 seconds
- Error spike < 5% during failover
- Full recovery within 60 seconds

**Abort Conditions**: Error rate > 10% or no recovery after 2 minutes

## Rollback Procedures

1. If failover fails: Manually promote replica
2. If application doesn't reconnect: Restart application pods
3. Emergency: Route traffic to backup region

## Post-Experiment

- [ ] Document observations
- [ ] Update runbooks with learnings
- [ ] Create tickets for improvements
- [ ] Schedule follow-up experiments
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

**1. Starting too big**

```
WRONG: First experiment kills an entire region
       Result: Major outage, chaos engineering gets banned

RIGHT: First experiment kills one instance of one service
       Result: Learn safely, build confidence, expand gradually
```

**2. No abort conditions**

```
WRONG: "Let's see what happens"
       Result: Experiment causes extended outage

RIGHT: "Abort if error rate > 5% or latency > 2s"
       Result: Experiment stops before causing real damage
```

**3. Running without monitoring**

```
WRONG: Run experiment, check results tomorrow
       Result: Miss the actual impact, can't correlate cause and effect

RIGHT: Watch dashboards in real-time during experiment
       Result: See exactly what happens, can abort if needed
```

**4. Not involving the team**

```
WRONG: One engineer runs chaos experiments secretly
       Result: Team panics, thinks it's a real incident

RIGHT: Announce experiments, involve the team
       Result: Learning opportunity, everyone understands the system better
```

### When NOT to Run Chaos Experiments

- During peak traffic periods (Black Friday, major events)
- When on-call engineer is unavailable
- During other maintenance windows
- When recent deployments haven't been validated
- When monitoring is degraded

---

## 8️⃣ When NOT to Do Chaos Engineering

### Situations Where It's Premature

1. **No monitoring**: Can't observe the impact
2. **No basic resilience**: No circuit breakers, retries, fallbacks
3. **Single instance**: Nothing to fail over to
4. **Early startup**: Focus on features first
5. **No incident response process**: Can't handle what you find

### Prerequisites for Chaos Engineering

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CHAOS ENGINEERING MATURITY                            │
│                                                                          │
│  Level 0: NOT READY                                                      │
│  ─────────────────────                                                   │
│  • No monitoring                                                         │
│  • No redundancy                                                         │
│  • No incident response                                                  │
│  → Focus on basics first                                                │
│                                                                          │
│  Level 1: READY FOR TESTING                                              │
│  ──────────────────────────                                              │
│  • Basic monitoring in place                                            │
│  • Some redundancy                                                       │
│  • Incident response process exists                                     │
│  → Run chaos experiments in test environment                            │
│                                                                          │
│  Level 2: READY FOR PRODUCTION                                           │
│  ─────────────────────────────                                           │
│  • Comprehensive monitoring                                              │
│  • Resilience patterns implemented                                      │
│  • Practiced incident response                                          │
│  → Run controlled production experiments                                │
│                                                                          │
│  Level 3: CONTINUOUS CHAOS                                               │
│  ──────────────────────────                                              │
│  • Automated chaos experiments                                          │
│  • Chaos is part of CI/CD                                               │
│  • Team expects and handles failures                                    │
│  → Chaos Monkey runs continuously                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 9️⃣ Comparison: Chaos Engineering Approaches

| Approach                 | Pros                             | Cons                        | Best For            |
| ------------------------ | -------------------------------- | --------------------------- | ------------------- |
| Manual GameDays          | Controlled, learning opportunity | Time-consuming, infrequent  | Starting out        |
| Automated (Chaos Monkey) | Continuous, catches drift        | Can cause unexpected issues | Mature systems      |
| CI/CD Integration        | Catches issues early             | Limited to test environment | All systems         |
| Production Chaos         | Most realistic                   | Highest risk                | Very mature systems |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is chaos engineering?**

A: Chaos engineering is the practice of intentionally injecting failures into a system to discover weaknesses before they cause real incidents. It's like a fire drill for software. Instead of hoping your failover works, you actually test it by killing servers, adding network latency, or simulating dependency failures. The goal is to build confidence that your system can handle real-world failures gracefully.

**Q: Why would you intentionally break production?**

A: Because production is the only environment that truly reflects reality. Test environments don't have real traffic patterns, real data volumes, or real infrastructure complexity. By running controlled experiments in production with proper safeguards (monitoring, abort conditions, small blast radius), you learn how the system actually behaves under failure. The alternative is finding out during a real incident when you're not prepared.

### L5 (Mid-Level) Questions

**Q: How would you design a chaos experiment for a microservices system?**

A: I'd follow a structured approach: (1) Define steady state: What metrics indicate "healthy"? (error rate, latency, throughput). (2) Form hypothesis: "If service X fails, the system will degrade gracefully due to circuit breakers." (3) Design experiment: What to break (kill one instance of service X), duration (5 minutes), abort conditions (error rate > 5%). (4) Prepare: Notify team, set up monitoring dashboards, verify rollback procedures. (5) Execute: Inject failure, observe behavior, record everything. (6) Analyze: Did hypothesis hold? What surprised us? (7) Improve: Fix weaknesses, update runbooks, schedule next experiment.

**Q: What's the difference between chaos engineering and testing?**

A: Traditional testing verifies known behaviors: "Given input X, output should be Y." Chaos engineering explores unknown behaviors: "What happens when the database is slow?" Testing is deterministic and repeatable. Chaos engineering is exploratory and often reveals surprises. Testing happens in controlled environments. Chaos engineering ideally happens in production. Testing proves things work. Chaos engineering proves things fail gracefully. They're complementary: testing ensures correctness, chaos engineering ensures resilience.

### L6 (Senior) Questions

**Q: How would you build a culture of chaos engineering in an organization?**

A: Building the culture is harder than the technology: (1) Start with leadership buy-in: Show the cost of past incidents that chaos engineering could have prevented. (2) Start small and safe: First experiments in non-production, then low-risk production experiments. (3) Make it visible: Share results widely, celebrate findings (even failures). (4) Blameless culture: Findings are opportunities, not finger-pointing. (5) Integrate into processes: Make resilience a deployment requirement. (6) GameDays: Regular team exercises build skills and confidence. (7) Automate gradually: Start manual, automate as confidence grows. (8) Measure progress: Track mean time to recovery, incident frequency, confidence scores. The goal is making failure a normal part of operations, not a crisis.

**Q: How do you balance chaos engineering with system stability?**

A: The key is controlled risk: (1) Blast radius: Start with one instance, one service, one region. Expand only after proving resilience at smaller scales. (2) Timing: Run during business hours when engineers are available. Avoid peak traffic, holidays, or during other changes. (3) Abort conditions: Define clear thresholds that automatically stop experiments. (4) Monitoring: Watch in real-time, don't run blind. (5) Rollback readiness: Have one-click rollback for every experiment. (6) Communication: Everyone knows what's happening and why. (7) Progressive confidence: Each successful experiment earns the right to larger experiments. The goal isn't to break things; it's to build confidence. If experiments regularly cause outages, you're doing it wrong.

---

## 1️⃣1️⃣ One Clean Mental Summary

Chaos engineering is fire drills for your software. You intentionally inject failures (kill servers, add latency, break dependencies) to discover weaknesses before real incidents expose them. The process: define what "healthy" looks like, hypothesize how the system should handle a failure, run a controlled experiment, observe results, and fix what you find. Start small (one instance), have abort conditions, and watch closely. Netflix runs Chaos Monkey continuously in production because they'd rather find problems on their terms than during a real crisis. The goal isn't breaking things; it's building confidence that your system can handle whatever the real world throws at it.
