# Notification System - Interview Grilling Q&A

## Trade-off Questions

### Q1: Why use Kafka instead of a simple queue like SQS/RabbitMQ?

**Answer:**

```
Kafka is chosen because:

1. Throughput:
   - Need 1M+ messages/second
   - Kafka designed for this scale
   - SQS: ~3K/sec per queue
   - RabbitMQ: ~50K/sec

2. Replay Capability:
   - Can replay messages for debugging
   - Useful for failed notification recovery
   - SQS deletes after consumption

3. Multiple Consumers:
   - Same message to analytics, webhooks, delivery
   - Consumer groups for scaling
   - SQS requires SNS fan-out

4. Ordering:
   - Per-user ordering with partition key
   - Important for notification sequence
   - SQS FIFO has throughput limits

When would I use SQS?
├── Lower scale (< 100K/sec)
├── Simpler operations
├── AWS-native integration
└── Don't need replay

Trade-offs of Kafka:
├── More complex to operate
├── Higher infrastructure cost
├── Requires ZooKeeper/KRaft
└── Learning curve
```

### Q2: How do you handle provider rate limits?

**Answer:**

```
Multiple strategies for rate limit handling:

1. Token Bucket Rate Limiter:
   - Track tokens per provider
   - Refill at provider's rate
   - Block when empty

2. Adaptive Rate Limiting:
   - Monitor 429 responses
   - Automatically reduce rate
   - Gradually increase after success

3. Multiple Accounts:
   - FCM: Multiple Firebase projects
   - Twilio: Multiple phone numbers
   - SendGrid: Multiple subusers

4. Queue Backpressure:
   - Slow down consumption when rate limited
   - Let queue absorb burst
   - Process at sustainable rate

Implementation:
```java
public class AdaptiveRateLimiter {
    private volatile double currentRate;
    private final double maxRate;
    
    public void onSuccess() {
        // Gradually increase rate
        currentRate = Math.min(maxRate, currentRate * 1.01);
    }
    
    public void onRateLimit() {
        // Immediately reduce rate
        currentRate = currentRate * 0.5;
        log.warn("Rate limited, reducing to {}/sec", currentRate);
    }
}
```

5. Priority Queues:
   - High priority gets rate limit allocation first
   - Low priority waits
   - Ensures critical notifications delivered
```

### Q3: Why separate queues per channel instead of one queue?

**Answer:**

```
Separate queues because:

1. Different Processing Speeds:
   - Push: 500K/sec (fast)
   - Email: 100K/sec (medium)
   - SMS: 60K/sec (slow)
   - One queue would bottleneck on slowest

2. Independent Scaling:
   - Scale push workers separately
   - SMS needs fewer workers
   - Cost optimization

3. Isolation:
   - SMS provider down doesn't affect push
   - Email bounce handling isolated
   - Easier debugging

4. Priority Handling:
   - Each channel has high/normal/low queues
   - 9 queues total (3 channels × 3 priorities)
   - Fine-grained control

Alternative: Single queue with routing
├── Simpler architecture
├── But: Head-of-line blocking
├── But: Can't scale channels independently
└── Works for smaller scale

For 10B notifications/day:
├── Separate queues essential
├── Isolation critical
└── Independent scaling required
```

### Q4: How do you ensure at-least-once delivery without spamming users?

**Answer:**

```
Balance reliability with user experience:

1. Idempotency Keys:
   - Client provides unique key per notification
   - Server deduplicates within 24-hour window
   - Safe to retry

2. Content-Based Deduplication:
   - Hash of (user_id, template_id, data)
   - 1-hour window
   - Prevents accidental duplicates

3. Collapse Keys (Push):
   - Same collapse key replaces previous
   - User sees only latest
   - Good for "new message" notifications

4. Provider-Level Dedup:
   - APNS/FCM have their own dedup
   - Email: Message-ID header
   - SMS: Provider dedup window

5. User-Level Rate Limiting:
   - Max 100 notifications/hour per user
   - Prevents runaway loops
   - Protects user experience

Implementation:
```java
public boolean shouldSend(Notification notification) {
    // Check idempotency
    if (!dedupService.checkIdempotency(notification.getIdempotencyKey())) {
        return false;
    }
    
    // Check content dedup
    if (dedupService.isDuplicate(notification)) {
        return false;
    }
    
    // Check rate limit
    if (!rateLimiter.tryAcquire(notification.getUserId())) {
        return false;
    }
    
    return true;
}
```
```

---

## Scaling Questions

### Q5: How would you handle 10x traffic (100B notifications/day)?

**Answer:**

```
Current: 10B/day, 1M/sec peak
Target: 100B/day, 10M/sec peak

Phase 1: Horizontal Scaling (Week 1-2)
├── Kafka: 50 → 200 brokers
├── Push workers: 200 → 1000
├── Email workers: 200 → 1000
├── PostgreSQL shards: 50 → 200
└── Cost: 5x increase

Phase 2: Provider Scaling (Week 2-4)
├── FCM: Add Firebase projects
├── SendGrid: Add dedicated IPs
├── Twilio: Add phone numbers/short codes
└── Negotiate volume discounts

Phase 3: Architecture Optimization (Month 2)
├── Regional deployment
├── Edge processing for in-app
├── Smarter batching
└── Reduce per-notification overhead

Phase 4: Cost Optimization (Month 3)
├── Reserved instances
├── Spot instances for workers
├── Provider negotiation
└── Target: 3x cost for 10x scale

Key Challenges:
1. Provider rate limits (need multiple accounts)
2. Database sharding (more complex queries)
3. Cost management (SMS especially)
4. Operational complexity
```

### Q6: What breaks first under load?

**Answer:**

```
Failure order during traffic spike:

1. Provider Rate Limits (First)
   - Symptom: 429 errors from APNS/FCM
   - Impact: Notifications queued
   - Fix: Backoff, multiple accounts

2. Kafka Consumer Lag
   - Symptom: Queue depth growing
   - Impact: Delayed notifications
   - Fix: Add consumers, increase partitions

3. Database Connections
   - Symptom: Connection pool exhausted
   - Impact: Token lookups fail
   - Fix: PgBouncer, increase pool

4. Redis Memory
   - Symptom: OOM, evictions
   - Impact: Rate limiting fails, duplicates
   - Fix: Add nodes, increase memory

5. Worker Memory
   - Symptom: OOM kills
   - Impact: Notifications lost (if not acked)
   - Fix: Tune batch sizes, add workers

Mitigation strategy:
├── Auto-scaling on all tiers
├── Circuit breakers for providers
├── Graceful degradation (drop low priority)
├── Load shedding during extreme spikes
```

### Q7: How do you handle a viral event (10x spike in 5 minutes)?

**Answer:**

```
Scenario: Breaking news, everyone gets push notification

Immediate (0-1 minute):
├── Queue depth spikes
├── Auto-scaling triggers
├── Rate limiters activate
└── Priority queues help

Short-term (1-5 minutes):
├── New workers spinning up
├── Provider rate limits hit
├── Backpressure to API
└── Return 429 to low-priority requests

Handling:
1. Pre-planned capacity:
   - Keep 2x headroom for spikes
   - Warm standby workers

2. Priority-based shedding:
   - High priority: Always send
   - Normal: Queue with delay
   - Low: Reject during spike

3. Smart batching:
   - Batch similar notifications
   - Reduce per-message overhead

4. Provider coordination:
   - Pre-notify providers of large sends
   - Request temporary rate increase

5. User communication:
   - "Notifications may be delayed"
   - Status page update

Post-spike:
├── Drain queues over 30-60 minutes
├── Process backlog by priority
├── Review and tune thresholds
```

---

## Failure Scenarios

### Q8: What happens if APNS is down for 1 hour?

**Answer:**

```
APNS outage handling:

Detection (0-1 minute):
├── Connection failures
├── Timeout errors
├── Circuit breaker opens
└── Alert triggered

Immediate Response:
├── Stop sending to APNS
├── Queue iOS notifications
├── Continue FCM/email/SMS
└── Log for retry

During Outage:
├── Kafka retains messages (24h retention)
├── Queue depth grows
├── Monitor queue size
└── No data loss

Recovery:
├── Circuit breaker half-open
├── Test with small batch
├── If success, resume
├── Process backlog gradually

User Impact:
├── iOS users: Delayed notifications
├── Android users: Unaffected
├── Email/SMS: Unaffected
├── Fallback: Send email if configured

Implementation:
```java
@CircuitBreaker(name = "apns", fallbackMethod = "apnsFallback")
public PushResult sendApns(PushNotification notification) {
    return apnsClient.send(notification);
}

public PushResult apnsFallback(PushNotification notification, Exception e) {
    // Queue for retry
    retryQueue.enqueue(notification, Duration.ofMinutes(5));
    
    // Try fallback channel if configured
    if (notification.hasFallbackChannel()) {
        return sendFallback(notification);
    }
    
    return PushResult.queued();
}
```
```

### Q9: How do you handle a bad template that sends wrong content?

**Answer:**

```
Scenario: Template bug sends "Hello {{name}}" literally

Detection:
├── QA catches before production
├── Automated template validation
├── User reports
├── Monitoring for template errors

Prevention:
1. Template Validation:
   - Syntax check on save
   - Required variable validation
   - Preview with test data

2. Staged Rollout:
   - New templates to 1% first
   - Monitor for errors
   - Full rollout after validation

3. Version Control:
   - All templates versioned
   - Rollback capability
   - Audit trail

Response to Incident:
1. Disable template immediately
2. Rollback to previous version
3. Assess blast radius
4. Notify affected users if severe

Implementation:
```java
public void updateTemplate(String templateId, TemplateUpdate update) {
    // Validate syntax
    templateValidator.validate(update.getContent());
    
    // Validate variables
    Set<String> variables = templateParser.extractVariables(update.getContent());
    if (!variables.containsAll(update.getRequiredVariables())) {
        throw new InvalidTemplateException("Missing required variables");
    }
    
    // Save as new version
    Template current = templateRepository.findById(templateId);
    Template newVersion = current.createNewVersion(update);
    templateRepository.save(newVersion);
    
    // Invalidate cache
    templateCache.invalidate(templateId);
}
```
```

### Q10: What if a user complains they're not receiving notifications?

**Answer:**

```
Debugging user notification issues:

Step 1: Check notification history
├── Query by user_id
├── Find recent notifications
├── Check status (sent, delivered, failed)

Step 2: Check device tokens
├── Are tokens registered?
├── Are tokens valid (not expired)?
├── Platform correct?

Step 3: Check user preferences
├── Channel enabled?
├── Category enabled?
├── In quiet hours?
├── Unsubscribed?

Step 4: Check delivery status
├── Provider response
├── Error codes
├── Bounce/failure reason

Step 5: Check provider
├── APNS: Token valid?
├── FCM: Token registered?
├── Email: Bounced? Spam?

Common Issues:
1. Invalid device token (app reinstalled)
2. User disabled notifications in OS
3. Email in spam folder
4. Category unsubscribed
5. Rate limited

Debugging API:
```http
GET /v1/debug/user/{user_id}/notifications
Response:
{
  "user_id": "user_123",
  "devices": [
    { "token": "***abc", "platform": "ios", "valid": true }
  ],
  "preferences": {
    "push_enabled": true,
    "email_enabled": true,
    "quiet_hours": false
  },
  "recent_notifications": [
    {
      "id": "notif_001",
      "status": "delivered",
      "channel": "push",
      "timestamp": "2024-01-15T10:30:00Z"
    }
  ]
}
```
```

---

## Design Evolution Questions

### Q11: What would you build first with 4 weeks?

**Answer:**

```
MVP (4 weeks, 4 engineers):

Week 1: Core Infrastructure
├── API endpoint (POST /notifications)
├── PostgreSQL schema
├── Basic template system
├── Device token registration

Week 2: Push Delivery
├── APNS integration
├── FCM integration
├── Basic retry logic
└── Delivery tracking

Week 3: Email + Preferences
├── SendGrid integration
├── User preferences API
├── Unsubscribe handling
└── Basic rate limiting

Week 4: Polish
├── Error handling
├── Basic monitoring
├── Documentation
└── Testing

What's NOT included:
├── SMS (expensive, add later)
├── In-app notifications
├── Kafka (use SQS)
├── Advanced analytics
├── A/B testing
├── Scheduling
├── Bulk/segment sends

This MVP:
├── Handles 10K notifications/day
├── Push + email only
├── Basic preferences
├── Foundation for growth
```

### Q12: How does the architecture evolve?

**Answer:**

```
Year 1: MVP to Product
├── 1M notifications/day
├── Push + Email
├── SQS for queuing
├── Single region
└── Cost: $10K/month

Year 2: Scale
├── 100M notifications/day
├── Add SMS
├── Migrate to Kafka
├── Add analytics
├── Multi-region
└── Cost: $100K/month

Year 3: Enterprise
├── 1B notifications/day
├── Add in-app
├── Advanced segmentation
├── A/B testing
├── SLA guarantees
└── Cost: $500K/month

Year 4: Platform
├── 10B notifications/day
├── Self-service portal
├── API marketplace
├── Custom integrations
├── Global presence
└── Cost: $5M/month

Key Architecture Changes:
├── Year 1→2: SQS to Kafka, add sharding
├── Year 2→3: Multi-region, analytics
├── Year 3→4: Self-service, platform
```

---

## Level-Specific Expectations

### L4 (Entry-Level)

Should demonstrate:
- Understanding of push notification basics
- Simple queue-based architecture
- Basic API design

May struggle with:
- Provider rate limiting
- Scale considerations
- Delivery guarantees

### L5 (Mid-Level)

Should demonstrate:
- Complete multi-channel design
- Kafka for queuing
- Rate limiting implementation
- Failure handling

May struggle with:
- 10B scale considerations
- Cost optimization
- Provider management

### L6 (Senior)

Should demonstrate:
- Multiple architecture options
- Deep provider knowledge
- Cost analysis
- Evolution over time
- Organizational considerations

---

## Summary: Key Points

1. **Multi-channel**: Push, email, SMS, in-app with unified API
2. **Kafka for scale**: 1M+ notifications/second
3. **Provider management**: Rate limits, failover, cost
4. **User preferences**: Respect opt-outs, quiet hours
5. **Deduplication**: Idempotency keys, content hash
6. **Priority queues**: Urgent vs marketing
7. **Observability**: Track delivery, opens, clicks
8. **Cost awareness**: SMS is expensive ($7.50/1000)

