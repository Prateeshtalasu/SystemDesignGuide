# 🐛 Debugging & On-Call

## 0️⃣ Prerequisites

Before diving into debugging and on-call, you should understand:

- **Logging**: How applications write logs (covered in Topic 10)
- **Metrics**: System measurements (covered in Topic 10)
- **Distributed Systems**: Services communicating over networks (Phase 1)
- **Java Basics**: Exceptions, stack traces, threads

Quick refresher on **stack traces**: When an error occurs, Java prints a stack trace showing the sequence of method calls that led to the error. This helps identify where the problem occurred.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain of Production Debugging

**Problem 1: The "3 AM Page"**

```
3:00 AM: PagerDuty alert: "High error rate"
3:05 AM: Engineer wakes up, checks dashboard
3:10 AM: Error rate is 5%, normally 0.1%
3:15 AM: Which errors? Which service? Which users?
3:20 AM: Start searching logs (millions of lines)
3:30 AM: Still searching...
3:45 AM: Found error pattern
4:00 AM: Identified root cause
4:15 AM: Fix deployed
4:30 AM: Service restored

Total downtime: 1.5 hours
Users affected: Thousands
```

**Problem 2: The "Intermittent Bug"**

```
Bug appears:
- Sometimes works, sometimes fails
- No clear pattern
- Can't reproduce locally
- Only happens in production
- Affects 0.1% of requests

Debugging:
- Check logs: Nothing obvious
- Check metrics: Nothing unusual
- Try to reproduce: Can't
- Wait for it to happen again
- Still can't find it
```

**Problem 3: The "Silent Failure"**

```
Service appears healthy:
- CPU: Normal
- Memory: Normal
- Error rate: Normal
- Response time: Normal

But:
- Users report issues
- Business metrics declining
- No alerts firing

Something is wrong, but nothing is alerting.
```

**Problem 4: The "Blame Game"**

```
Issue occurs.
Team meeting:
- "It's the database"
- "No, it's the API"
- "No, it's the load balancer"
- "No, it's the network"

Everyone points fingers.
No one knows the real cause.
```

### What Breaks Without Good Debugging

| Scenario | Without Good Debugging | With Good Debugging |
|----------|----------------------|-------------------|
| Time to resolution | Hours/days | Minutes |
| Root cause | Unknown/guessed | Identified |
| Reproducibility | Can't reproduce | Can reproduce |
| Learning | Repeat same issues | Learn and prevent |
| User impact | High (long downtime) | Low (quick resolution) |

---

## 2️⃣ Intuition and Mental Model

### The Detective Analogy

Debugging is like being a **detective solving a crime**.

**Without good tools**:
- No evidence (logs)
- No witnesses (metrics)
- No crime scene photos (traces)
- Guesswork and hunches

**With good tools**:
- Evidence (structured logs with correlation IDs)
- Witnesses (metrics showing what happened)
- Crime scene photos (distributed traces)
- Systematic investigation

### Debugging Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    DEBUGGING PROCESS                             │
│                                                                  │
│  1. DETECT                                                       │
│     Alert fires or user reports issue                           │
│                                                                  │
│  2. OBSERVE                                                      │
│     Check dashboards, logs, metrics                             │
│     Understand current state                                    │
│                                                                  │
│  3. HYPOTHESIZE                                                 │
│     "Maybe it's the database?"                                  │
│     "Maybe it's a recent deployment?"                          │
│                                                                  │
│  4. INVESTIGATE                                                 │
│     Check logs for that service                                 │
│     Check metrics for anomalies                                 │
│     Check traces for slow requests                              │
│                                                                  │
│  5. VERIFY                                                       │
│     Confirm hypothesis with evidence                            │
│                                                                  │
│  6. FIX                                                          │
│     Deploy fix or rollback                                      │
│                                                                  │
│  7. VERIFY FIX                                                   │
│     Confirm issue is resolved                                   │
│                                                                  │
│  8. DOCUMENT                                                     │
│     Write postmortem                                            │
│     Update runbooks                                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ Reading Stack Traces

### Understanding Stack Traces

```
Exception in thread "main" java.lang.NullPointerException
    at com.example.PaymentService.processPayment(PaymentService.java:45)
    at com.example.PaymentController.process(PaymentController.java:23)
    at com.example.Application.main(Application.java:10)
```

**Reading from bottom to top**:
1. `Application.main` - Entry point
2. `PaymentController.process` - Called from main
3. `PaymentService.processPayment` - Called from controller
4. Line 45 - Where the error occurred

**The error**: `NullPointerException` at line 45 of `PaymentService.java`

### Common Exceptions

**NullPointerException**:
```java
String name = user.getName();  // user is null
int length = name.length();     // NPE here
```

**ArrayIndexOutOfBoundsException**:
```java
int[] arr = {1, 2, 3};
int value = arr[5];  // Index 5 doesn't exist
```

**ClassCastException**:
```java
Object obj = "hello";
Integer num = (Integer) obj;  // Can't cast String to Integer
```

**IllegalArgumentException**:
```java
public void setAge(int age) {
    if (age < 0) {
        throw new IllegalArgumentException("Age cannot be negative");
    }
}
```

### Debugging with Stack Traces

```java
// BAD: Swallowing exceptions
try {
    processPayment(request);
} catch (Exception e) {
    // Silent failure - no information
}

// GOOD: Logging with context
try {
    processPayment(request);
} catch (PaymentException e) {
    logger.error("Payment processing failed",
        kv("paymentId", request.getPaymentId()),
        kv("userId", request.getUserId()),
        kv("amount", request.getAmount()),
        e);  // Include stack trace
    throw e;
}
```

---

## 4️⃣ Thread Dumps and Heap Dumps

### Thread Dumps

**What they show**: State of all threads at a point in time

**When to use**: 
- Application appears hung
- High CPU usage
- Deadlocks
- Thread pool exhaustion

**How to generate**:
```bash
# Get process ID
jps

# Generate thread dump
jstack <pid> > thread-dump.txt

# Or kill -3 <pid> (sends SIGQUIT)
```

**Reading thread dumps**:
```
"http-nio-8080-exec-1" #23 daemon prio=5 os_prio=0 tid=0x00007f8b8c123000 nid=0x1234 runnable [0x00007f8b9c456000]
   java.lang.Thread.State: RUNNABLE
        at java.net.SocketInputStream.socketRead0(Native Method)
        at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
        at java.net.SocketInputStream.read(SocketInputStream.java:171)
        at java.net.SocketInputStream.read(SocketInputStream.java:141)
        at org.apache.catalina.connector.InputBuffer.read(InputBuffer.java:330)
        at org.apache.catalina.connector.CoyoteInputStream.read(CoyoteInputStream.java:183)
```

**What to look for**:
- Threads in `BLOCKED` state (potential deadlock)
- Many threads waiting on same lock
- Threads stuck in `RUNNABLE` (high CPU)
- Thread pool threads all busy (exhaustion)

### Heap Dumps

**What they show**: Memory contents at a point in time

**When to use**:
- OutOfMemoryError
- Memory leaks
- High memory usage

**How to generate**:
```bash
# Generate heap dump on OOM
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof MyApp

# Generate manually
jmap -dump:format=b,file=heap.hprof <pid>
```

**Analyzing with Eclipse MAT or VisualVM**:
- Find largest objects
- Find memory leaks (objects that shouldn't be retained)
- Analyze object references

---

## 5️⃣ Root Cause Analysis: 5 Whys

### The 5 Whys Technique

Ask "why" five times to get to root cause.

**Example**:

```
Problem: Payment service is down

Why? Payment service crashed
Why? Out of memory error
Why? Memory leak in payment processing
Why? Payment objects not being garbage collected
Why? Payment objects held in static cache that never clears

Root cause: Static cache never evicts entries
Solution: Add TTL or size limit to cache
```

### Structured RCA Process

```
1. Define the problem
   "Payment service error rate is 5%"

2. Gather data
   - Error logs
   - Metrics (CPU, memory, latency)
   - Recent changes (deployments, config)
   - User reports

3. Identify symptoms
   - Errors: "Database connection timeout"
   - Pattern: Affects 5% of requests
   - Timing: Started 2 hours ago

4. Form hypotheses
   - Database is overloaded
   - Connection pool exhausted
   - Network issue

5. Test hypotheses
   - Check database metrics: CPU 90%
   - Check connection pool: 100% utilized
   - Check network: Normal

6. Identify root cause
   "Database connection pool exhausted due to slow queries"

7. Identify contributing factors
   - Slow query introduced in recent deployment
   - Connection pool size too small
   - No query timeout

8. Propose solutions
   - Rollback recent deployment
   - Increase connection pool size
   - Add query timeout
   - Fix slow query

9. Implement fix
10. Verify fix
11. Document (postmortem)
```

---

## 6️⃣ Incident Response

### Incident Severity Levels

**P0 (Critical)**:
- Service completely down
- Data loss
- Security breach
- All hands on deck

**P1 (High)**:
- Major feature broken
- Significant user impact
- Degraded performance
- Response within 1 hour

**P2 (Medium)**:
- Minor feature broken
- Limited user impact
- Response within 4 hours

**P3 (Low)**:
- Cosmetic issues
- Minimal impact
- Response within 1 business day

### Incident Response Process

```
1. DETECT
   Alert fires or user reports

2. ACKNOWLEDGE
   On-call engineer acknowledges
   Creates incident ticket

3. ASSESS
   Determine severity
   Check impact (users, revenue)

4. COMMUNICATE
   Update status page
   Notify stakeholders
   Create Slack channel

5. INVESTIGATE
   Follow runbook if available
   Check logs, metrics, traces
   Identify root cause

6. MITIGATE
   Quick fix or workaround
   Rollback if needed
   Scale up resources

7. RESOLVE
   Permanent fix deployed
   Service restored
   Verify with monitoring

8. POSTMORTEM
   Document what happened
   Identify improvements
   Update runbooks
```

### Runbook Example

```markdown
# Runbook: Payment Service High Error Rate

## Symptoms
- Error rate > 1%
- Alert: PaymentErrorRateHigh
- Users reporting payment failures

## Quick Checks
1. Check service status: `kubectl get pods -n payment`
2. Check error logs: `kubectl logs -f deployment/payment-service | grep ERROR`
3. Check metrics: Grafana dashboard "Payment Service"

## Common Causes

### Database Connection Issues
**Symptoms**: "Connection timeout" errors
**Fix**:
1. Check database status: `kubectl get pods -n database`
2. Check connection pool metrics
3. Restart payment service if needed

### External API Down
**Symptoms**: "Payment gateway timeout"
**Fix**:
1. Check payment gateway status page
2. Enable fallback payment method if available
3. Contact payment gateway support

### Recent Deployment
**Symptoms**: Errors started after deployment
**Fix**:
1. Check deployment time vs error start time
2. Rollback: `kubectl rollout undo deployment/payment-service`
3. Verify rollback: Check error rate

## Escalation
- If not resolved in 15 minutes: Escalate to payment team lead
- If P0: Page on-call manager

## Resolution Verification
- Error rate < 0.1%
- No user complaints for 30 minutes
- All systems green
```

---

## 7️⃣ Postmortems

### Postmortem Structure

```markdown
# Postmortem: Payment Service Outage - 2024-01-15

## Summary
Payment service was down for 45 minutes, affecting 10,000 users.

## Timeline
- 14:30 - Deployment of v2.1.0
- 14:35 - First errors reported
- 14:40 - Error rate reached 50%
- 14:45 - On-call engineer paged
- 15:00 - Root cause identified
- 15:15 - Rollback completed
- 15:15 - Service restored

## Impact
- Duration: 45 minutes
- Users affected: 10,000
- Revenue impact: $50,000
- Severity: P0

## Root Cause
Database connection pool exhausted due to connection leak in v2.1.0.

## Contributing Factors
1. Connection leak bug in new code
2. Insufficient integration tests
3. No canary deployment (deployed to 100% immediately)
4. Connection pool size too small for traffic

## What Went Well
- Quick detection (5 minutes)
- Fast rollback (15 minutes)
- Good communication

## What Went Wrong
- Bug not caught in testing
- No gradual rollout
- Insufficient monitoring of connection pool

## Action Items
- [ ] Fix connection leak (assigned: Alice, due: 2024-01-20)
- [ ] Add connection pool monitoring (assigned: Bob, due: 2024-01-18)
- [ ] Implement canary deployments (assigned: Charlie, due: 2024-01-25)
- [ ] Add integration test for connection handling (assigned: Alice, due: 2024-01-22)

## Lessons Learned
- Always use canary deployments for database-related changes
- Monitor connection pool metrics
- Integration tests should cover connection lifecycle
```

### Blameless Postmortems

**Key principles**:
- Focus on process, not people
- Learn from mistakes
- Improve systems, not blame individuals
- Everyone makes mistakes

**Bad postmortem**:
- "Alice deployed bad code"
- "Bob didn't test properly"
- "Charlie should have caught this"

**Good postmortem**:
- "Deployment process didn't catch the bug"
- "Testing didn't cover this scenario"
- "Monitoring didn't alert early enough"

---

## 8️⃣ On-Call Rotations

### Rotation Models

**Primary/Secondary**:
- Primary: First responder
- Secondary: Backup if primary unavailable
- Rotate weekly

**Follow-the-Sun**:
- Different time zones
- Handoff at end of day
- 24/7 coverage

**On-Demand**:
- No fixed schedule
- Page when needed
- For smaller teams

### On-Call Best Practices

1. **Clear escalation path**
2. **Well-documented runbooks**
3. **Access to all systems**
4. **Communication channels** (Slack, PagerDuty)
5. **Post-incident rest** (no meetings after major incident)
6. **Compensation** (on-call pay, time off)

---

## 9️⃣ Interview Follow-Up Questions

### Q1: "How do you debug a production issue?"

**Answer**:
Systematic approach:

1. **Gather information**: Check alerts, dashboards, user reports. Understand symptoms.

2. **Check recent changes**: Recent deployments, config changes, traffic patterns.

3. **Examine logs**: Search for errors around the time issue started. Use correlation IDs to trace requests.

4. **Check metrics**: Look for anomalies (CPU, memory, error rate, latency).

5. **Use distributed traces**: If available, trace a failing request through all services.

6. **Form hypothesis**: Based on evidence, form a hypothesis about root cause.

7. **Test hypothesis**: Check logs/metrics that would confirm or refute hypothesis.

8. **Fix or mitigate**: Deploy fix or rollback. Verify resolution.

9. **Document**: Write postmortem, update runbooks.

Key: Be systematic, use data (logs/metrics/traces), don't guess.

### Q2: "What's in your debugging toolkit?"

**Answer**:
Essential tools:

1. **Log aggregation**: ELK, Splunk, or similar for searching logs
2. **Metrics dashboards**: Grafana, Datadog for visualizing metrics
3. **Distributed tracing**: Jaeger, Zipkin for request traces
4. **APM tools**: New Relic, Datadog APM for application performance
5. **Command line**: kubectl, docker, jstack, jmap
6. **Database tools**: Query analyzers, connection pool monitors
7. **Network tools**: tcpdump, wireshark for network issues

For Java specifically: jstack (thread dumps), jmap (heap dumps), VisualVM, Eclipse MAT.

### Q3: "How do you handle an incident when you're on-call?"

**Answer**:
Process:

1. **Acknowledge**: Respond to alert immediately (within SLA).

2. **Assess severity**: Determine if it's P0, P1, etc. Check user impact.

3. **Communicate**: Update status page, notify team, create incident channel.

4. **Follow runbook**: If runbook exists, follow it. If not, start investigation.

5. **Investigate**: Check logs, metrics, recent changes. Use 5 Whys technique.

6. **Mitigate**: Quick fix or rollback to restore service. Don't wait for perfect fix.

7. **Verify**: Confirm service is restored, metrics are normal.

8. **Document**: After resolution, document what happened and what was done.

9. **Postmortem**: Schedule postmortem within 48 hours.

Key: Restore service first (mitigate), then find root cause. Don't let perfect be the enemy of good.

### Q4: "How do you prevent alert fatigue?"

**Answer**:
Strategies:

1. **Tune alert thresholds**: Only alert on actionable issues. Avoid noise.

2. **Use alert severity**: P0 for critical, P3 for informational. Filter by severity.

3. **Alert grouping**: Group related alerts. Don't send 100 alerts for same issue.

4. **Runbooks**: Every alert should have a runbook. If no runbook, alert might not be needed.

5. **Review and remove**: Regularly review alerts. Remove ones that never require action.

6. **On-call rotation**: Share burden. Don't have one person always on-call.

7. **Alert testing**: Test that alerts fire when they should, don't fire when they shouldn't.

8. **SLO-based alerts**: Alert on SLO violations, not every metric spike.

### Q5: "What makes a good postmortem?"

**Answer**:
Good postmortem includes:

1. **Timeline**: What happened when, in chronological order.

2. **Impact**: Users affected, duration, business impact.

3. **Root cause**: Not just symptoms, but actual root cause (use 5 Whys).

4. **Contributing factors**: What made the issue worse or harder to detect.

5. **What went well**: What worked during incident response.

6. **What went wrong**: What didn't work, what could be better.

7. **Action items**: Specific, assigned, with due dates. Track to completion.

8. **Lessons learned**: What we learned, how we'll prevent this.

Key principles: Blameless (focus on process, not people), actionable (specific improvements), timely (within 48 hours).

---

## 🔟 One Clean Mental Summary

Debugging production issues requires systematic investigation using logs, metrics, and traces. Start with symptoms, check recent changes, examine logs for errors, use distributed traces to follow request flow, form hypotheses based on evidence, and verify before fixing.

On-call requires clear processes: acknowledge alerts, assess severity, communicate, investigate using runbooks, mitigate quickly (rollback if needed), verify resolution, and document in postmortems. Use 5 Whys for root cause analysis. Postmortems should be blameless and focus on improving systems.

The key insight: Good debugging is systematic, not guesswork. Use observability tools (logs, metrics, traces) to gather evidence, form hypotheses, and verify. On-call is about restoring service quickly, then finding root cause. Learn from incidents to prevent recurrence.






