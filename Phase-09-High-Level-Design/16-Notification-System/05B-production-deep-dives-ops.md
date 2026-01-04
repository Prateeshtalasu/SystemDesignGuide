# Notification System - Production Deep Dives (Operations)

## 1. Scaling Strategy

### Horizontal Scaling Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Scaling Architecture                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  API Layer (Stateless, auto-scale on requests):                              │
│  ├── API Gateway: 120 instances                                              │
│  ├── Notification Service: 240 instances                                     │
│  └── Scale trigger: CPU > 60% or requests > 10K/sec/instance                │
│                                                                               │
│  Worker Layer (Scale with queue depth):                                       │
│  ├── Push Workers: 200 instances (scale on queue lag)                        │
│  ├── Email Workers: 200 instances                                            │
│  ├── SMS Workers: 100 instances                                              │
│  └── In-App Workers: 50 instances                                            │
│                                                                               │
│  Queue Layer (Kafka - scale with partitions):                                 │
│  ├── Brokers: 50                                                             │
│  ├── Partitions: 2000 total                                                  │
│  └── Scale: Add brokers, rebalance partitions                                │
│                                                                               │
│  Data Layer:                                                                  │
│  ├── PostgreSQL: 228 instances (sharded)                                     │
│  ├── Redis: 50 nodes                                                         │
│  └── ClickHouse: 20 nodes                                                    │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Auto-Scaling Configuration

```yaml
# Kubernetes HPA for Push Workers
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: push-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: push-worker
  minReplicas: 50
  maxReplicas: 500
  metrics:
  - type: External
    external:
      metric:
        name: kafka_consumer_lag
        selector:
          matchLabels:
            topic: push-queue-normal
      target:
        type: AverageValue
        averageValue: 10000  # Scale when lag > 10K per pod
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

### Provider Rate Limit Handling

```java
@Service
public class ProviderRateLimiter {
    
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    @PostConstruct
    public void init() {
        // APNS: ~100K/sec per connection, 100 connections
        limiters.put("apns", RateLimiter.create(10_000_000));
        
        // FCM: 240K/min per project, multiple projects
        limiters.put("fcm", RateLimiter.create(400_000));
        
        // SendGrid: 100K/sec with dedicated IPs
        limiters.put("sendgrid", RateLimiter.create(100_000));
        
        // Twilio: 100/sec per number, 600 numbers
        limiters.put("twilio", RateLimiter.create(60_000));
    }
    
    public boolean tryAcquire(String provider) {
        RateLimiter limiter = limiters.get(provider);
        return limiter != null && limiter.tryAcquire();
    }
    
    public void acquire(String provider) throws InterruptedException {
        RateLimiter limiter = limiters.get(provider);
        if (limiter != null) {
            limiter.acquire();
        }
    }
    
    // Adaptive rate limiting based on provider responses
    public void adjustRate(String provider, double factor) {
        RateLimiter current = limiters.get(provider);
        double newRate = current.getRate() * factor;
        limiters.put(provider, RateLimiter.create(newRate));
        
        log.info("Adjusted {} rate to {}/sec", provider, newRate);
    }
}
```

---

## 2. Reliability Patterns

### Circuit Breaker for Providers

```java
@Service
public class ResilientPushService {
    
    private final CircuitBreaker apnsCircuitBreaker;
    private final CircuitBreaker fcmCircuitBreaker;
    
    public ResilientPushService() {
        this.apnsCircuitBreaker = CircuitBreaker.of("apns", CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .slidingWindowSize(100)
            .permittedNumberOfCallsInHalfOpenState(10)
            .build());
        
        this.fcmCircuitBreaker = CircuitBreaker.of("fcm", CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .slidingWindowSize(100)
            .build());
        
        // Monitor state changes
        apnsCircuitBreaker.getEventPublisher()
            .onStateTransition(event -> {
                log.warn("APNS circuit breaker: {}", event.getStateTransition());
                alertService.sendAlert("APNS circuit breaker: " + event.getStateTransition());
            });
    }
    
    public PushResult sendPush(PushNotification notification) {
        if (notification.getPlatform() == Platform.IOS) {
            return apnsCircuitBreaker.executeSupplier(() -> apnsService.send(notification));
        } else {
            return fcmCircuitBreaker.executeSupplier(() -> fcmService.send(notification));
        }
    }
}
```

### Retry with Exponential Backoff

```java
@Service
public class RetryableNotificationService {
    
    private final RetryTemplate retryTemplate;
    
    public RetryableNotificationService() {
        this.retryTemplate = RetryTemplate.builder()
            .maxAttempts(5)
            .exponentialBackoff(1000, 2, 60000) // 1s, 2s, 4s, 8s, 16s (max 60s)
            .retryOn(RetryableException.class)
            .retryOn(TimeoutException.class)
            .build();
    }
    
    public DeliveryResult sendWithRetry(Notification notification) {
        return retryTemplate.execute(context -> {
            int attempt = context.getRetryCount() + 1;
            log.debug("Attempt {} for notification {}", attempt, notification.getId());
            
            try {
                return send(notification);
            } catch (ProviderUnavailableException e) {
                // Retryable
                throw new RetryableException(e);
            } catch (InvalidTokenException e) {
                // Not retryable
                throw e;
            }
        }, context -> {
            // Recovery callback after all retries exhausted
            log.error("All retries exhausted for {}", notification.getId());
            return DeliveryResult.failed("Max retries exceeded");
        });
    }
}
```

### Fallback Channels

```java
@Service
public class FallbackNotificationService {
    
    public void sendWithFallback(NotificationRequest request) {
        List<Channel> channels = request.getChannels();
        Channel fallback = request.getFallbackChannel();
        
        boolean anyDelivered = false;
        
        for (Channel channel : channels) {
            try {
                DeliveryResult result = send(request, channel);
                if (result.isSuccess()) {
                    anyDelivered = true;
                }
            } catch (Exception e) {
                log.warn("Failed to send via {}: {}", channel, e.getMessage());
            }
        }
        
        // If no channel succeeded and fallback is configured
        if (!anyDelivered && fallback != null && !channels.contains(fallback)) {
            log.info("Using fallback channel {} for {}", fallback, request.getId());
            try {
                send(request, fallback);
            } catch (Exception e) {
                log.error("Fallback channel {} also failed", fallback, e);
            }
        }
    }
}
```

---

## 3. Monitoring and Observability

### Key Metrics

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Key Metrics                                         │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Throughput Metrics:                                                          │
│  ├── notifications_received_total (counter by channel)                       │
│  ├── notifications_sent_total (counter by channel, provider)                 │
│  ├── notifications_delivered_total (counter by channel)                      │
│  └── notifications_failed_total (counter by channel, error_type)             │
│                                                                               │
│  Latency Metrics:                                                             │
│  ├── notification_processing_latency_ms (histogram)                          │
│  ├── notification_delivery_latency_ms (histogram by channel)                 │
│  ├── provider_request_latency_ms (histogram by provider)                     │
│  └── queue_wait_time_ms (histogram by queue)                                 │
│                                                                               │
│  Queue Metrics:                                                               │
│  ├── kafka_consumer_lag (gauge by topic, partition)                          │
│  ├── kafka_messages_in_rate (gauge by topic)                                 │
│  └── queue_depth (gauge by priority)                                         │
│                                                                               │
│  Provider Metrics:                                                            │
│  ├── provider_success_rate (gauge by provider)                               │
│  ├── provider_error_rate (gauge by provider, error_code)                     │
│  ├── provider_rate_limit_hits (counter by provider)                          │
│  └── invalid_token_rate (gauge by platform)                                  │
│                                                                               │
│  Business Metrics:                                                            │
│  ├── delivery_rate (gauge by channel)                                        │
│  ├── open_rate (gauge by channel, template)                                  │
│  ├── click_rate (gauge by channel, template)                                 │
│  └── unsubscribe_rate (gauge by channel)                                     │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Alerting Rules

```yaml
groups:
- name: notification-alerts
  rules:
  
  # Delivery rate drop
  - alert: LowDeliveryRate
    expr: |
      (sum(rate(notifications_delivered_total[5m])) / 
       sum(rate(notifications_sent_total[5m]))) < 0.9
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Delivery rate below 90%"
      
  # High queue lag
  - alert: HighKafkaLag
    expr: |
      sum(kafka_consumer_lag) by (topic) > 100000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kafka consumer lag > 100K messages"
      
  # Provider errors
  - alert: HighProviderErrorRate
    expr: |
      (sum(rate(notifications_failed_total{error_type="provider"}[5m])) /
       sum(rate(notifications_sent_total[5m]))) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Provider error rate > 5%"
      
  # Invalid tokens spike
  - alert: InvalidTokenSpike
    expr: |
      rate(invalid_tokens_total[5m]) > 1000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High rate of invalid tokens"
      
  # Circuit breaker open
  - alert: CircuitBreakerOpen
    expr: |
      circuit_breaker_state{state="open"} == 1
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Circuit breaker {{ $labels.name }} is OPEN"
```

### Distributed Tracing

```java
@Service
public class TracedNotificationService {
    
    private final Tracer tracer;
    
    public NotificationResult process(NotificationRequest request) {
        Span span = tracer.spanBuilder("process-notification")
            .setAttribute("notification.id", request.getId())
            .setAttribute("notification.channels", String.join(",", request.getChannels()))
            .setAttribute("notification.priority", request.getPriority().name())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // Validate
            Span validateSpan = tracer.spanBuilder("validate").startSpan();
            validate(request);
            validateSpan.end();
            
            // Check preferences
            Span prefsSpan = tracer.spanBuilder("check-preferences").startSpan();
            UserPreferences prefs = preferencesService.get(request.getUserId());
            prefsSpan.setAttribute("preferences.cached", prefs.isCached());
            prefsSpan.end();
            
            // Rate limit
            Span rateLimitSpan = tracer.spanBuilder("rate-limit").startSpan();
            boolean allowed = rateLimiter.tryAcquire(request.getUserId());
            rateLimitSpan.setAttribute("rate_limit.allowed", allowed);
            rateLimitSpan.end();
            
            if (!allowed) {
                span.setAttribute("result", "rate_limited");
                return NotificationResult.rateLimited();
            }
            
            // Enqueue
            Span enqueueSpan = tracer.spanBuilder("enqueue").startSpan();
            enqueue(request);
            enqueueSpan.end();
            
            span.setAttribute("result", "queued");
            return NotificationResult.queued(request.getId());
            
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

---

## 4. Security Deep Dive

### API Authentication

```java
@Component
public class ApiKeyAuthFilter extends OncePerRequestFilter {
    
    private final ApiKeyService apiKeyService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain) throws ServletException, IOException {
        
        String apiKey = request.getHeader("Authorization");
        
        if (apiKey == null || !apiKey.startsWith("Bearer ")) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.getWriter().write("{\"error\": \"Missing API key\"}");
            return;
        }
        
        String key = apiKey.substring(7);
        
        try {
            ApiKeyInfo keyInfo = apiKeyService.validate(key);
            
            // Set security context
            SecurityContextHolder.getContext().setAuthentication(
                new ApiKeyAuthentication(keyInfo)
            );
            
            // Add to MDC for logging
            MDC.put("client_id", keyInfo.getClientId());
            MDC.put("api_key_id", keyInfo.getKeyId());
            
            filterChain.doFilter(request, response);
            
        } catch (InvalidApiKeyException e) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.getWriter().write("{\"error\": \"Invalid API key\"}");
        } finally {
            MDC.clear();
        }
    }
}
```

### Data Privacy

```java
@Service
public class PrivacyService {
    
    // Mask PII in logs
    public String maskEmail(String email) {
        if (email == null) return null;
        int atIndex = email.indexOf('@');
        if (atIndex <= 1) return "***@***";
        return email.charAt(0) + "***" + email.substring(atIndex);
    }
    
    public String maskPhone(String phone) {
        if (phone == null || phone.length() < 4) return "***";
        return "***" + phone.substring(phone.length() - 4);
    }
    
    // Handle GDPR deletion
    public void deleteUserData(String userId) {
        // Delete device tokens
        deviceTokenRepository.deleteByUserId(userId);
        
        // Delete preferences
        preferencesRepository.deleteByUserId(userId);
        
        // Anonymize notification history (keep for analytics)
        notificationRepository.anonymize(userId);
        
        // Clear from cache
        cacheService.invalidateUser(userId);
        
        log.info("Deleted user data for {}", maskUserId(userId));
    }
    
    // Handle unsubscribe
    public void handleUnsubscribe(String email, String channel, String reason) {
        // Add to suppression list
        suppressionRepository.save(new Suppression(
            email,
            channel,
            reason,
            Instant.now()
        ));
        
        // Update user preferences if user exists
        userRepository.findByEmail(email).ifPresent(user -> {
            preferencesService.unsubscribe(user.getId(), channel);
        });
        
        // Audit log
        auditLog.log(AuditEvent.builder()
            .type("UNSUBSCRIBE")
            .email(maskEmail(email))
            .channel(channel)
            .reason(reason)
            .build());
    }
}
```

## 4. Failure Handling & Recovery

### Failure Scenario: Database Primary Failure

**Detection:**
- How to detect: Grafana alert "PostgreSQL primary down"
- Alert thresholds: > 10% error rate for 30 seconds
- Monitoring: Check PostgreSQL primary health check endpoint

**Impact:**
- Affected services: Device token storage, user preferences
- User impact: Notifications continue (read from replica)
- Degradation: New device registrations fail until recovery

**Recovery Steps:**
1. **T+0s**: Alert received, database connection failures detected
2. **T+10s**: Verify primary database status (AWS RDS console)
3. **T+20s**: Confirm primary failure (not just network issue)
4. **T+30s**: Trigger automatic failover to synchronous replica
5. **T+45s**: Replica promoted to primary
6. **T+60s**: Update application connection strings
7. **T+90s**: Applications reconnect to new primary
8. **T+120s**: Verify write operations successful
9. **T+150s**: Normal operations resume

**RTO:** < 3 minutes
**RPO:** < 1 second (synchronous replication)

**Prevention:**
- Multi-AZ deployment with synchronous replication
- Automated failover enabled
- Regular failover testing (quarterly)

### Failure Scenario: Push Notification Provider Outage

**Detection:**
- How to detect: Grafana alert "Push provider error rate > 50%"
- Alert thresholds: > 50% error rate for 1 minute
- Monitoring: Check push provider API status

**Impact:**
- Affected services: Push notification delivery
- User impact: Push notifications fail, fallback to email/SMS
- Degradation: System continues operating with alternative channels

**Recovery Steps:**
1. **T+0s**: Alert received, push provider errors detected
2. **T+30s**: Verify provider status (APNs/FCM status page)
3. **T+60s**: If provider outage confirmed, enable fallback channels
4. **T+90s**: Route push notifications to email/SMS
5. **T+120s**: Monitor provider recovery
6. **T+300s**: Provider recovers, resume push notifications
7. **T+330s**: Normal operations resume

**RTO:** < 5 minutes
**RPO:** 0 (notifications queued, delivered via fallback)

**Prevention:**
- Multiple push providers (APNs + FCM)
- Automatic fallback to email/SMS
- Health checks every 30 seconds

### Failure Scenario: Email Service Outage

**Detection:**
- How to detect: Grafana alert "Email provider error rate > 50%"
- Alert thresholds: > 50% error rate for 2 minutes
- Monitoring: Check email provider API status

**Impact:**
- Affected services: Email notification delivery
- User impact: Email notifications fail, fallback to push/SMS
- Degradation: System continues operating with alternative channels

**Recovery Steps:**
1. **T+0s**: Alert received, email provider errors detected
2. **T+30s**: Verify email provider status (SendGrid/Mailgun dashboard)
3. **T+60s**: If provider outage confirmed, enable fallback channels
4. **T+90s**: Route email notifications to push/SMS
5. **T+120s**: Monitor provider recovery
6. **T+600s**: Provider recovers, resume email notifications
7. **T+630s**: Normal operations resume

**RTO:** < 10 minutes
**RPO:** 0 (notifications queued, delivered via fallback)

**Prevention:**
- Multiple email providers (primary + backup)
- Automatic failover between providers
- Health checks every 30 seconds

### Failure Scenario: Kafka Lag (Notification Queue Backup)

**Detection:**
- How to detect: Grafana alert "Kafka consumer lag > 10M messages"
- Alert thresholds: Lag > 10M messages for 10 minutes
- Monitoring: Check Kafka consumer lag metrics

**Impact:**
- Affected services: Notification processing pipeline
- User impact: Notifications delayed (queued but not sent)
- Degradation: System continues operating, slower processing

**Recovery Steps:**
1. **T+0s**: Alert received, Kafka lag detected
2. **T+30s**: Check consumer group status, identify slow consumers
3. **T+60s**: Scale up notification workers (add 50 more pods)
4. **T+120s**: New pods join consumer group, begin processing
5. **T+300s**: Lag begins decreasing
6. **T+600s**: Lag returns to normal (< 100K messages)

**RTO:** < 10 minutes
**RPO:** 0 (messages in Kafka, no loss)

**Prevention:**
- Auto-scaling based on Kafka lag
- Monitor consumer processing rate
- Alert on lag thresholds

---

## 4.5. Simulation (End-to-End User Journeys)

### Journey 1: User Receives Notification (Happy Path)

**Step-by-step:**

1. **Trigger**: Order placed event published to Kafka
2. **Notification Service**: 
   - Consumes event from Kafka
   - Fetches user preferences
   - Selects channel (push, email, SMS)
   - Renders template
3. **Delivery**: 
   - Push: Sent to device token
   - Email: Sent via email service
   - SMS: Sent via SMS provider
4. **Tracking**: 
   - Notification status tracked
   - Delivery confirmed
5. **Response**: User receives notification within 5 seconds

### Journey 2: Multi-Channel Notification

**Step-by-step:**

1. **Trigger**: Critical alert (payment failed)
2. **Notification Service**: 
   - Priority: HIGH
   - Channels: Push + Email + SMS (all channels)
   - Templates: Different for each channel
3. **Delivery**: 
   - Push: Immediate (real-time)
   - Email: Sent via queue (5s delay acceptable)
   - SMS: Sent immediately (critical)
4. **Response**: User receives via all 3 channels within 10 seconds

### High-Load / Contention Scenario: Breaking News Notification Storm

```
Scenario: Breaking news event triggers millions of notifications
Time: Major event (peak traffic)

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Breaking news event triggers 10M notifications        │
│ - Event: "Breaking: Major announcement"                    │
│ - Users: 10M subscribers to news channel                   │
│ - Expected: ~100K notifications/minute normal traffic      │
│ - 100x traffic spike                                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+0-30min: Notification Processing Storm                  │
│ - 10M notifications to process                            │
│ - Channels: Push (80%), Email (15%), SMS (5%)            │
│ - Processing rate: 100K notifications/minute              │
│ - Total time: 100 minutes (within acceptable)            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Load Handling:                                             │
│                                                              │
│ 1. Kafka Buffering:                                       │
│    - 10M events published to Kafka                       │
│    - Kafka partitions: 100 (even distribution)            │
│    - Each partition: 100K events                         │
│    - Kafka buffers: 7-day retention (safe)               │
│                                                              │
│ 2. Notification Processing:                              │
│    - Consumers: 100 instances (one per partition)        │
│    - Each consumer: 1K notifications/minute              │
│    - Processing: Validate → Template → Deliver           │
│    - Latency: 2-5 seconds per notification              │
│                                                              │
│ 3. Channel-Specific Handling:                            │
│    - Push: 8M notifications (fast, async)                │
│      * FCM/APNS: Handles 100K/second (safe)             │
│      * Processing time: 80 minutes                      │
│    - Email: 1.5M notifications (queued)                  │
│      * Email service: 10K/minute (SES limit)            │
│      * Processing time: 150 minutes (acceptable)        │
│    - SMS: 500K notifications (queued)                    │
│      * SMS provider: 1K/minute (rate limit)             │
│      * Processing time: 500 minutes (acceptable queue)   │
│                                                              │
│ 4. Rate Limiting:                                        │
│    - Per-user rate limit: 10 notifications/hour         │
│    - Prevents notification spam                         │
│    - Users receive max 10 notifications (throttled)     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - All 10M notifications queued successfully              │
│ - Push notifications delivered within 80 minutes        │
│ - Email notifications delivered within 150 minutes      │
│ - SMS notifications delivered within 500 minutes        │
│ - Rate limiting prevents spam                            │
│ - No service degradation                                   │
└─────────────────────────────────────────────────────────────┘
```

### Edge Case Scenario: Duplicate Notification Prevention

```
Scenario: Same event triggers multiple notifications to same user
Edge Case: Event deduplication and idempotency

┌─────────────────────────────────────────────────────────────┐
│ T+0ms: Order placed event published                      │
│ - Event ID: event_001                                     │
│ - User ID: user_123                                      │
│ - Event: OrderCreated{order_id: "order_456"}            │
│ - Published to Kafka: notification-events topic          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+100ms: Consumer 1 processes event                     │
│ - Consumer: notification-consumer-group-1                │
│ - Processes: event_001 for user_123                     │
│ - Sends notification: Push notification sent            │
│ - Status: DELIVERED                                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+200ms: Event republished (retry or duplicate)        │
│ - Event ID: event_001 (same event, duplicate)          │
│ - User ID: user_123                                      │
│ - Published again to Kafka (system retry)              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+300ms: Consumer 2 processes same event                │
│ - Consumer: notification-consumer-group-2               │
│ - Processes: event_001 for user_123 (duplicate)        │
│ - Problem: User receives duplicate notification         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Edge Case: Duplicate Notification Prevention            │
│                                                              │
│ Problem: Same event processed twice                      │
│ - User receives 2 identical notifications               │
│ - Poor user experience (notification spam)             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Solution: Idempotency Keys and Deduplication            │
│                                                              │
│ 1. Idempotency Key:                                      │
│    - Each notification has idempotency key              │
│    - Key: hash(event_id + user_id + channel + template)│
│    - Store: notification_idempotency:{key}             │
│    - TTL: 24 hours (prevent duplicates)                │
│                                                              │
│ 2. Deduplication Check:                                 │
│    - Before sending: Check Redis for idempotency key   │
│    - If exists: Skip (already sent)                    │
│    - If not exists: Send and store key                │
│                                                              │
│ 3. Database Unique Constraint:                          │
│    - notifications table: UNIQUE(event_id, user_id, channel)│
│    - Prevents duplicate inserts                        │
│    - Database enforces idempotency at storage layer     │
│                                                              │
│ 4. Event Deduplication:                                 │
│    - At source: Event ID tracked in Kafka              │
│    - Consumer: Track processed event IDs               │
│    - If event already processed: Skip                  │
│    - Kafka offset tracking prevents reprocessing       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - Duplicate events detected and skipped                 │
│ - User receives notification exactly once              │
│ - No notification spam                                  │
│ - System idempotent at all layers                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Cost Analysis

### Monthly Cost Breakdown

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                      Monthly Cost Breakdown                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Compute:                                                                     │
│  ├── API/Service (360 instances × $150)    = $54,000                        │
│  ├── Push Workers (200 × $100)             = $20,000                        │
│  ├── Email Workers (200 × $100)            = $20,000                        │
│  ├── SMS Workers (100 × $50)               = $5,000                         │
│  ├── In-App Workers (50 × $100)            = $5,000                         │
│  ├── WebSocket Servers (600 × $200)        = $120,000                       │
│  └── Subtotal:                             = $224,000                        │
│                                                                               │
│  Databases:                                                                   │
│  ├── PostgreSQL (228 × $300)               = $68,400                        │
│  ├── Redis (50 × $200)                     = $10,000                        │
│  ├── ClickHouse (20 × $500)                = $10,000                        │
│  └── Subtotal:                             = $88,400                         │
│                                                                               │
│  Messaging:                                                                   │
│  ├── Kafka (50 × $400)                     = $20,000                        │
│  └── Subtotal:                             = $20,000                         │
│                                                                               │
│  Third-Party Providers:                                                       │
│  ├── Push (APNS/FCM): Free                 = $0                             │
│  ├── Email (SendGrid): 2.5B × $0.0001      = $250,000                       │
│  ├── SMS (Twilio): 500M × $0.0075          = $3,750,000                     │
│  └── Subtotal:                             = $4,000,000                      │
│                                                                               │
│  Network:                                                                     │
│  ├── Data transfer                         = $150,000                       │
│  ├── Load balancers                        = $20,000                        │
│  └── Subtotal:                             = $170,000                        │
│                                                                               │
│  Other:                                                                       │
│  ├── Monitoring                            = $50,000                        │
│  ├── Storage                               = $10,000                        │
│  └── Subtotal:                             = $60,000                         │
│                                                                               │
│  TOTAL:                                    = $4,562,400/month                │
│                                                                               │
│  Cost per 1000 notifications:                                                 │
│  ├── Push: $0.002                                                            │
│  ├── Email: $0.10                                                            │
│  ├── SMS: $7.50                                                              │
│  ├── In-App: $0.002                                                          │
│  └── Blended: $0.46                                                          │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Cost Optimization Strategies

**Current Monthly Cost:** $2,000,000
**Target Monthly Cost:** $1,400,000 (30% reduction)

**Top 3 Cost Drivers:**
1. **SMS Costs:** $1,000,000/month (50%) - Largest single component
2. **Email Costs:** $500,000/month (25%) - Email provider costs
3. **Compute (EC2):** $200,000/month (10%) - Application servers

**Optimization Strategies (Ranked by Impact):**

1. **SMS Optimization (50% savings):**
   - **Current:** High SMS volume for all notifications
   - **Optimization:** 
     - Use push as primary for non-critical notifications
     - Batch SMS for daily digests
     - Negotiate volume discounts
   - **Savings:** $500,000/month (50% of $1M SMS)
   - **Trade-off:** Slight delay for batched SMS (acceptable)

2. **Email Provider Optimization (20% savings):**
   - **Current:** Single email provider
   - **Optimization:** 
     - Use SES for transactional (cheaper)
     - SendGrid for marketing (better deliverability)
     - Warm up IPs gradually
   - **Savings:** $100,000/month (20% of $500K email)
   - **Trade-off:** Slight complexity increase (acceptable)

3. **Reduce Notification Volume (20% savings across channels):**
   - **Current:** High notification volume
   - **Optimization:** 
     - Smart batching
     - User preference optimization
     - Suppress inactive users
   - **Savings:** $300,000/month (20% of $1.5M total channels)
   - **Trade-off:** Slight reduction in engagement (acceptable)

4. **Compute Optimization (30% savings):**
   - **Current:** On-demand instances
   - **Optimization:** 
     - Spot instances for workers
     - Right-size instances
     - Reserved capacity for baseline
   - **Savings:** $60,000/month (30% of $200K compute)
   - **Trade-off:** Possible interruptions for spot instances (acceptable)

**Total Potential Savings:** $960,000/month (48% reduction)
**Optimized Monthly Cost:** $1,040,000/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| SMS Notification | $0.01 | $0.005 | 50% |
| Email Notification | $0.001 | $0.0008 | 20% |
| Push Notification | $0.0001 | $0.00008 | 20% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** SMS Optimization → $500K savings
2. **Phase 2 (Month 2):** Email Optimization, Reduce Volume → $400K savings
3. **Phase 3 (Month 3):** Compute Optimization → $60K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor notification delivery rates (ensure no degradation)
- Monitor SMS/email costs (target < $600K/month)
- Review and adjust quarterly

---

## 6. Backup and Recovery

**RPO (Recovery Point Objective):** < 1 minute
- Maximum acceptable data loss
- Based on database replication lag (1 minute max)
- Device tokens and user preferences replicated with < 1 second lag
- Notification logs replicated within 1 hour (acceptable for analytics)

**RTO (Recovery Time Objective):** < 5 minutes
- Maximum acceptable downtime
- Time to restore service via failover to DR region
- Includes database failover, cache warm-up, and traffic rerouting

**Backup Strategy:**
- **Database Backups**: Daily full backups, hourly incremental
- **Point-in-time Recovery**: Retention period of 7 days
- **Cross-region Replication**: Streaming replication to DR region
- **Kafka Mirroring**: Real-time mirroring to DR region (RPO = 0)
- **ClickHouse Backups**: Daily backups to S3

**Restore Steps:**
1. Detect primary region failure (health checks) - 30 seconds
2. Promote database replica to primary - 1 minute

### Database Connection Pool Configuration

**Connection Pool Settings (HikariCP):**

```yaml
spring:
  datasource:
    hikari:
      # Pool sizing
      maximum-pool-size: 20          # Max connections per application instance
      minimum-idle: 5                 # Minimum idle connections
      
      # Timeouts
      connection-timeout: 30000       # 30s - Max time to wait for connection from pool
      idle-timeout: 600000            # 10m - Idle connection timeout
      max-lifetime: 1800000           # 30m - Max connection lifetime
      
      # Monitoring
      leak-detection-threshold: 60000 # 60s - Detect connection leaks
      
      # Validation
      connection-test-query: SELECT 1
      validation-timeout: 3000        # 3s - Connection validation timeout
      
      # Register metrics
      register-mbeans: true
```

**Pool Sizing Calculation:**

```
Pool size = ((Core count × 2) + Effective spindle count)

For our service (228 PostgreSQL instances, sharded):
- 4 CPU cores per pod
- Spindle count = 1 per instance
- Calculated: (4 × 2) + 1 = 9 connections minimum
- With safety margin: 20 connections per pod
- 240 pods × 20 = 4800 max connections distributed across 228 instances
- Per-instance average: ~21 connections per instance
```

**Pool Exhaustion Mitigation:**

1. **Monitoring:**
   - Alert when pool usage > 80%
   - Track wait time for connections
   - Monitor connection leak detection

2. **Circuit Breaker:**
   - If pool exhausted for > 5 seconds, open circuit breaker
   - Fail fast instead of blocking

3. **PgBouncer (Connection Pooler):**
   - Place PgBouncer between app and PostgreSQL instances
   - PgBouncer pool: 40 connections per instance
   - App pools can share PgBouncer connections efficiently
3. Update DNS records to point to DR region - 30 seconds
4. Warm up Redis cache from database - 2 minutes
5. Verify notification service health and resume traffic - 1 minute

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Database replication lag < 1 minute
- [ ] Kafka replication lag < 1 minute
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture notification delivery success rate
   - Document active device token count
   - Verify database replication lag (< 1 minute)
   - Test notification sending (100 sample notifications)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+5 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-2 min:** Promote secondary region to primary
   - **T+2-3 min:** Update DNS records (Route53 health checks)
   - **T+3-4 min:** Rebalance Kafka partitions
   - **T+4-4.5 min:** Warm up cache from database
   - **T+4.5-5 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+5 to T+15 minutes):**
   - Verify RTO < 5 minutes: ✅ PASS/FAIL
   - Verify RPO < 1 minute: Check database replication lag at failure time
   - Test notification sending: Send 1,000 test notifications, verify >99% success
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify notification delivery: Check delivery rates match baseline

5. **Data Integrity Verification:**
   - Compare device token count: Pre-failover vs post-failover (should match within 1 min)
   - Spot check: Verify 100 random device tokens accessible
   - Check notification history: Verify notification logs intact

6. **Failback Procedure (T+15 to T+20 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag < 1 minute
   - Update DNS to route traffic back to primary
   - Monitor for 5 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 5 minutes: Time from failure to service resumption
- ✅ RPO < 1 minute: Maximum data loss (verified via database replication lag)
- ✅ Notification delivery works: >99% notifications delivered
- ✅ No data loss: Device token count matches pre-failover (within 1 min window)
- ✅ Service resumes within RTO target: All metrics return to baseline

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: User Receives Push Notification

**Step-by-step:**

1. **Event Trigger**: User's friend posts a photo, triggers notification event
2. **Notification Service**: 
   - Receives event: `{"user_id": "user123", "type": "friend_post", "data": {...}}`
   - Fetches user preferences: `GET user_preferences:user123` → Push enabled
   - Fetches device tokens: `SELECT token FROM device_tokens WHERE user_id = 'user123'` → 2 tokens
   - Creates notification records: `INSERT INTO notifications (id, user_id, type, content, status)`
   - Publishes to Kafka: `{"notification_id": "notif_abc123", "user_id": "user123", "tokens": [...], "payload": {...}}`
3. **Push Worker** (consumes Kafka):
   - Receives notification message
   - For each device token:
     - Validates token (checks if device still active)
     - Sends to FCM/APNS: `POST https://fcm.googleapis.com/fcm/send`
     - Updates status: `UPDATE notifications SET status = 'sent' WHERE id = 'notif_abc123'`
4. **Device receives notification**:
   - FCM/APNS delivers to device
   - Device displays notification
5. **User opens notification**:
   - Client sends: `PUT /v1/notifications/notif_abc123/read`
   - Updates status: `UPDATE notifications SET status = 'read', read_at = NOW()`
6. **Result**: Notification delivered and read within 2 seconds

**Total latency: ~2 seconds** (Kafka 100ms + Push worker 500ms + FCM/APNS 1.4s)

### Journey 2: Batch Notification (Marketing Campaign)

**Step-by-step:**

1. **Marketing Action**: Admin creates campaign via `POST /v1/campaigns` targeting 10M users
2. **Campaign Service**: 
   - Creates campaign: `INSERT INTO campaigns (id, name, target_users, status)`
   - Publishes to Kafka: `{"campaign_id": "camp_xyz789", "user_ids": [10M users], "message": "..."}`
3. **Notification Service** (consumes Kafka):
   - Receives campaign message
   - Batches users (1000 per batch)
   - For each batch:
     - Fetches device tokens for batch
     - Creates notification records
     - Publishes to Kafka (notification queue)
4. **Push Workers** (consume notification queue):
   - Process notifications in parallel (200 workers)
   - Send to FCM/APNS
   - Update status
5. **Result**: 10M notifications delivered within 5 minutes

**Total time: ~5 minutes** (batch processing + parallel workers)

### Failure & Recovery Walkthrough

**Scenario: FCM Service Degradation**

**RTO (Recovery Time Objective):** < 5 minutes (automatic retry with exponential backoff)  
**RPO (Recovery Point Objective):** 0 (notifications queued, no data loss)

**Timeline:**

```
T+0s:    FCM starts returning 503 errors (throttling)
T+0-10s: Push notifications fail
T+10s:   Circuit breaker opens (after 5 consecutive failures)
T+10s:   Notifications queued to dead letter queue (Kafka)
T+15s:   Retry with exponential backoff (1s, 2s, 4s, 8s)
T+30s:   FCM recovers, accepts requests
T+35s:   Circuit breaker closes (after 3 successful requests)
T+40s:   Dead letter queue processed, notifications retried
T+5min:  All queued notifications delivered
```

**What degrades:**
- Push notifications delayed by 10-30 seconds
- Notifications queued in DLQ (no data loss)
- Email and SMS unaffected (different channels)

**What stays up:**
- Email notifications
- SMS notifications
- In-app notifications
- Notification creation and storage

**What recovers automatically:**
- Circuit breaker closes after FCM recovery
- Queued notifications processed automatically
- No manual intervention required

**What requires human intervention:**
- Investigate FCM throttling root cause
- Review capacity planning
- Consider FCM quota increases

**Cascading failure prevention:**
- Circuit breaker prevents retry storms
- Dead letter queue prevents notification loss
- Exponential backoff reduces FCM load
- Timeouts prevent thread exhaustion

## 7. Disaster Recovery

### Backup Strategy

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Backup Strategy                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Device Tokens (PostgreSQL):                                                  │
│  ├── Streaming replication to DR region                                      │
│  ├── Point-in-time recovery (7 days)                                         │
│  ├── Daily snapshots (30 days retention)                                     │
│  └── RPO: < 1 second                                                         │
│                                                                               │
│  User Preferences (PostgreSQL):                                               │
│  ├── Same as device tokens                                                   │
│  └── RPO: < 1 second                                                         │
│                                                                               │
│  Templates (PostgreSQL):                                                      │
│  ├── Version controlled in Git                                               │
│  ├── Database backup                                                         │
│  └── Can rebuild from Git                                                    │
│                                                                               │
│  Notification Log (ClickHouse):                                               │
│  ├── Replicated across 3 nodes                                               │
│  ├── Daily backups to S3                                                     │
│  └── RPO: < 1 hour                                                           │
│                                                                               │
│  Kafka:                                                                       │
│  ├── Replication factor 3                                                    │
│  ├── Cross-DC mirroring                                                      │
│  └── RPO: 0 (no message loss)                                                │
│                                                                               │
│  Redis:                                                                       │
│  ├── No backup (cache only)                                                  │
│  ├── Warm up from PostgreSQL                                                 │
│  └── Recovery: < 5 minutes                                                   │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Failover Procedure

**RTO (Recovery Time Objective):** < 5 minutes (manual failover to DR region)  
**RPO (Recovery Point Objective):** < 1 minute (async replication lag)

```
SCENARIO: Primary region failure

DETECTION (T+0):
1. Health checks fail
2. Multiple services unreachable
3. Alert triggered

ASSESSMENT (T+2 min):
1. Confirm region-wide outage
2. Check DR region health
3. Initiate failover

FAILOVER (T+5 min):
1. Update DNS to DR region
2. Promote PostgreSQL standby
3. Start Kafka consumers in DR
4. Scale up DR compute

VALIDATION (T+15 min):
1. Verify API availability
2. Test notification delivery
3. Check provider connectivity
4. Monitor error rates

IMPACT:
├── Queued notifications: Delayed ~15 min
├── In-flight notifications: May be duplicated (idempotent)
├── Data loss: None
└── Total downtime: ~15 minutes
```

