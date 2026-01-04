# Notification System - Production Deep Dives (Core)

## 1. Kafka Deep Dive: Message Queue

### Why Kafka for Notifications?

```
Problem: Need reliable, high-throughput message delivery

Requirements:
├── 1M+ notifications/second peak
├── At-least-once delivery
├── Message ordering per user
├── Multi-consumer support
├── Replay capability

Kafka provides:
├── High throughput (millions/sec)
├── Durable storage (7 days retention)
├── Consumer groups for scaling
├── Partition-based ordering
└── Exactly-once semantics (with transactions)
```

### Topic Architecture

```java
@Configuration
public class KafkaTopicConfig {
    
    @Bean
    public NewTopic notificationRequestsTopic() {
        return TopicBuilder.name("notification-requests")
            .partitions(100)
            .replicas(3)
            .config(TopicConfig.RETENTION_MS_CONFIG, "86400000") // 24 hours
            .config(TopicConfig.MIN_IN_SYNC_REPLICAS_CONFIG, "2")
            .build();
    }
    
    @Bean
    public NewTopic pushQueueHighTopic() {
        return TopicBuilder.name("push-queue-high")
            .partitions(200)
            .replicas(3)
            .config(TopicConfig.RETENTION_MS_CONFIG, "86400000")
            .build();
    }
    
    @Bean
    public NewTopic pushQueueNormalTopic() {
        return TopicBuilder.name("push-queue-normal")
            .partitions(200)
            .replicas(3)
            .config(TopicConfig.RETENTION_MS_CONFIG, "86400000")
            .build();
    }
    
    @Bean
    public NewTopic deliveryEventsTopic() {
        return TopicBuilder.name("delivery-events")
            .partitions(100)
            .replicas(3)
            .config(TopicConfig.RETENTION_MS_CONFIG, "604800000") // 7 days
            .build();
    }
}
```

### Producer Configuration

```java
@Configuration
public class KafkaProducerConfig {
    
    @Bean
    public ProducerFactory<String, Notification> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // High throughput settings
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 32768); // 32KB
        config.put(ProducerConfig.LINGER_MS_CONFIG, 5); // Wait 5ms for batching
        config.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864); // 64MB
        
        // Reliability settings
        config.put(ProducerConfig.ACKS_CONFIG, "all"); // Wait for all replicas
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        // Compression
        config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");
        
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

### Consumer with Retry

```java
@Service
public class PushWorkerConsumer {
    
    private final PushNotificationService pushService;
    private final KafkaTemplate<String, Notification> kafka;
    
    @KafkaListener(
        topics = "push-queue-normal",
        groupId = "push-workers",
        concurrency = "50",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void consume(
            @Payload PushNotification notification,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {
        
        try {
            // Process notification
            DeliveryResult result = pushService.send(notification);
            
            // Publish delivery event
            publishDeliveryEvent(notification, result);
            
            // Acknowledge
            ack.acknowledge();
            
        } catch (RetryableException e) {
            // Retry with backoff
            if (notification.getAttempts() < MAX_RETRIES) {
                scheduleRetry(notification);
                ack.acknowledge(); // Don't reprocess same message
            } else {
                // Max retries exceeded
                publishFailureEvent(notification, e);
                ack.acknowledge();
            }
        } catch (NonRetryableException e) {
            // Don't retry (invalid token, etc.)
            publishFailureEvent(notification, e);
            ack.acknowledge();
        }
    }
    
    private void scheduleRetry(PushNotification notification) {
        notification.incrementAttempts();
        int delayMs = calculateBackoff(notification.getAttempts());
        
        // Send to retry topic with delay
        kafka.send("push-retry", notification.getUserId(), notification)
            .addCallback(
                result -> log.debug("Scheduled retry for {}", notification.getId()),
                ex -> log.error("Failed to schedule retry", ex)
            );
    }
    
    private int calculateBackoff(int attempts) {
        // Exponential backoff: 1s, 2s, 4s, 8s, 16s
        return (int) Math.pow(2, attempts - 1) * 1000;
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Kafka Technology-Level

**Normal Flow: Notification Request Processing**

```
Step 1: Notification Request Received
┌─────────────────────────────────────────────────────────────┐
│ POST /v1/notifications                                       │
│ Request: {                                                    │
│   "user_id": "user_123",                                     │
│   "type": "order_shipped",                                   │
│   "priority": "high",                                         │
│   "data": {...}                                              │
│ }                                                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Notification Service Processes
┌─────────────────────────────────────────────────────────────┐
│ Notification Service:                                        │
│ 1. Validate request                                         │
│ 2. Check user preferences                                    │
│ 3. Check rate limits                                         │
│ 4. Check deduplication                                       │
│ 5. Publish to Kafka:                                         │
│    Topic: notification-requests                              │
│    Partition: hash(user_123) % 100 = partition 23            │
│    Message: {                                                │
│      "notification_id": "notif_001",                        │
│      "user_id": "user_123",                                 │
│      "type": "order_shipped",                               │
│      "priority": "high",                                     │
│      "data": {...}                                           │
│    }                                                          │
│ Latency: ~10ms                                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Push Worker Consumes
┌─────────────────────────────────────────────────────────────┐
│ Consumer Group: push-workers                                 │
│ Worker assigned to partition 23 polls:                     │
│ - Receives batch of 100 notifications                       │
│ - Filters by priority: High priority first                 │
│ - Checks device tokens (Redis)                              │
│ - Sends to APNS/FCM                                         │
│ - Publishes delivery events                                 │
│ Latency: ~50ms per notification                              │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Kafka Consumer Lag**

```
Scenario: Push service falls behind during traffic spike

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Normal operation (10K notifications/second)           │
│ T+60s: Breaking news event (1M notifications/second)        │
│ T+61s: Consumers can only process 100K/second              │
│ T+62s: Lag builds: 900K notifications/second                │
│ T+300s: Lag reaches 270M notifications                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Mitigation:                                                  │
│ 1. Auto-scale consumers: 10 → 100 instances                │
│ 2. Increase batch size: 100 → 1,000 notifications          │
│ 3. Priority processing: High priority first                │
│ 4. Result: Throughput increases to 1M/second               │
│ 5. Lag clears within 5 minutes                              │
└─────────────────────────────────────────────────────────────┘
```

**Idempotency Handling:**

```
Problem: Same notification processed twice (consumer retry)

Solution: Notification ID deduplication
1. Each notification has unique notification_id
2. Consumer tracks processed IDs (Redis set)
3. Duplicate notifications (same ID) ignored
4. Delivery to providers is idempotent (same token + payload)

Example:
- Notification processed twice → Same notification_id → Deduplicated → Idempotent
```

**Traffic Spike Handling:**

```
Scenario: Breaking news triggers millions of notifications

┌─────────────────────────────────────────────────────────────┐
│ Normal: 10K notifications/second                            │
│ Spike: 1M notifications/second (breaking news)            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Kafka Buffering:                                             │
│ - 1M notifications/second buffered in Kafka                 │
│ - 24-hour retention = 86B notifications capacity            │
│ - No message loss                                            │
│                                                              │
│ Consumer Scaling:                                            │
│ - Auto-scale: 10 → 100 consumer instances                  │
│ - Each consumer: 10K notifications/second                    │
│ - Throughput: 1M notifications/second                        │
│                                                              │
│ Priority Handling:                                           │
│ - High priority: Processed immediately                       │
│ - Normal priority: Processed after high priority            │
│                                                              │
│ Result:                                                      │
│ - Notifications queued in Kafka (no loss)                   │
│ - High priority delivered within seconds                    │
│ - Normal priority delivered within minutes                   │
└─────────────────────────────────────────────────────────────┘
```

### Hot Partition Mitigation

**Problem:** If a popular user receives many notifications, one Kafka partition could become hot, creating a bottleneck for notification processing.

**Example Scenario:**
```
Normal user: 10 notifications/hour → Partition 12 (hash(user_id) % partitions)
Popular user: 1,000 notifications/hour → Partition 12 (same partition!)
Result: Partition 12 receives 100x traffic, becomes bottleneck
Notification consumers for partition 12 become overwhelmed
```

**Detection:**
- Monitor partition lag per partition (Kafka metrics)
- Alert when partition lag > 2x average
- Track partition throughput metrics (notifications/second per partition)
- Monitor consumer lag per partition
- Alert threshold: Lag > 2x average for 5 minutes

**Mitigation Strategies:**

1. **Partition Key Strategy:**
   - Partition by `user_id` hash (distributes evenly across users)
   - For very active users, consider composite key (user_id + notification_type)
   - This distributes high-volume user notifications across multiple partitions

2. **Partition Rebalancing:**
   - If hot partition detected, increase partition count
   - Rebalance existing partitions (Kafka supports this)
   - Distribute hot user notifications across multiple partitions

3. **Consumer Scaling:**
   - Add more consumers for hot partitions
   - Auto-scale consumers based on partition lag
   - Each consumer handles subset of hot partition

4. **Priority Queue Strategy:**
   - Use priority queue for high-volume users
   - Separate high-priority notifications from normal notifications
   - Process high-priority notifications with dedicated consumers

5. **Rate Limiting:**
   - Rate limit per user (prevent single user from overwhelming)
   - Throttle high-volume users
   - Buffer notifications locally if rate limit exceeded

**Implementation:**

```java
@Service
public class HotPartitionDetector {
    
    public void detectAndMitigate() {
        Map<Integer, Long> partitionLag = getPartitionLag();
        double avgLag = partitionLag.values().stream()
            .mapToLong(Long::longValue)
            .average()
            .orElse(0);
        
        partitionLag.entrySet().stream()
            .filter(e -> e.getValue() > avgLag * 2)
            .forEach(e -> {
                int partition = e.getKey();
                long lag = e.getValue();
                
                // Alert ops team
                alertOps("Hot partition detected: " + partition + ", lag: " + lag);
                
                // Auto-scale consumers for this partition
                scaleConsumers(partition, lag);
            });
    }
}
```

**Monitoring:**
- Partition lag per partition (Kafka metrics)
- Partition throughput (notifications/second per partition)
- Consumer lag per partition
- Alert threshold: Lag > 2x average for 5 minutes

**Recovery Behavior:**

```
Auto-healing:
- Consumer crash → Kafka rebalance → Notifications reassigned → Retry
- Network timeout → Exponential backoff → Retry
- Processing failure → Dead letter queue → Alert ops

Human intervention:
- Persistent lag → Scale consumers or optimize processing
- Dead letter queue growth → Investigate and fix processing logic
- Provider failures → Switch to backup provider or retry later
```

### Dead Letter Queue

```java
@Configuration
public class DeadLetterConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Notification> 
            kafkaListenerContainerFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, Notification> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(AckMode.MANUAL);
        
        // Configure error handler with DLQ
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate,
                (record, ex) -> new TopicPartition(
                    record.topic() + "-dlq",
                    record.partition()
                )),
            new FixedBackOff(1000L, 3L) // 3 retries, 1s between
        ));
        
        return factory;
    }
}

// DLQ processor
@Service
public class DLQProcessor {
    
    @KafkaListener(topics = "push-queue-normal-dlq", groupId = "dlq-processor")
    public void processDLQ(
            @Payload PushNotification notification,
            @Header(KafkaHeaders.DLT_EXCEPTION_MESSAGE) String errorMessage) {
        
        log.error("DLQ message: {} - Error: {}", notification.getId(), errorMessage);
        
        // Alert operations
        alertService.sendAlert(
            "Notification DLQ",
            "Notification " + notification.getId() + " failed: " + errorMessage
        );
        
        // Store for manual review
        dlqRepository.save(new DLQRecord(notification, errorMessage));
    }
}
```

---

## 2. Redis Deep Dive: Caching and Rate Limiting

### Caching Strategy

```java
@Service
public class NotificationCacheService {
    
    private final RedisTemplate<String, Object> redis;
    
    // Device token cache
    public List<DeviceToken> getDeviceTokens(String userId) {
        String key = "tokens:" + userId;
        
        List<DeviceToken> cached = (List<DeviceToken>) redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // Cache miss - load from DB
        List<DeviceToken> tokens = deviceTokenRepository.findByUserId(userId);
        redis.opsForValue().set(key, tokens, Duration.ofHours(1));
        
        return tokens;
    }
    
    // User preferences cache
    public UserPreferences getPreferences(String userId) {
        String key = "prefs:" + userId;
        
        UserPreferences cached = (UserPreferences) redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        UserPreferences prefs = preferencesRepository.findByUserId(userId)
            .orElse(UserPreferences.defaults());
        redis.opsForValue().set(key, prefs, Duration.ofMinutes(15));
        
        return prefs;
    }
    
    // Template cache
    public NotificationTemplate getTemplate(String templateId) {
        String key = "template:" + templateId;
        
        NotificationTemplate cached = (NotificationTemplate) redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        NotificationTemplate template = templateRepository.findById(templateId)
            .orElseThrow(() -> new TemplateNotFoundException(templateId));
        redis.opsForValue().set(key, template, Duration.ofHours(24));
        
        return template;
    }
    
    // Invalidation
    public void invalidateTokens(String userId) {
        redis.delete("tokens:" + userId);
    }
    
    public void invalidatePreferences(String userId) {
        redis.delete("prefs:" + userId);
    }
    
    public void invalidateTemplate(String templateId) {
        redis.delete("template:" + templateId);
    }
}
```

### Rate Limiting Implementation

```java
@Service
public class RateLimiter {
    
    private final RedisTemplate<String, String> redis;
    
    // Sliding window rate limiter
    public boolean tryAcquire(String userId, String channel, int limit, Duration window) {
        String key = "ratelimit:" + userId + ":" + channel;
        long now = System.currentTimeMillis();
        long windowStart = now - window.toMillis();
        
        // Use Redis sorted set for sliding window
        return redis.execute(new SessionCallback<Boolean>() {
            @Override
            public Boolean execute(RedisOperations operations) {
                operations.multi();
                
                // Remove old entries
                operations.opsForZSet().removeRangeByScore(key, 0, windowStart);
                
                // Count current entries
                operations.opsForZSet().count(key, windowStart, now);
                
                // Add new entry
                operations.opsForZSet().add(key, UUID.randomUUID().toString(), now);
                
                // Set expiry
                operations.expire(key, window.plusMinutes(1));
                
                List<Object> results = operations.exec();
                Long count = (Long) results.get(1);
                
                return count < limit;
            }
        });
    }
    
    // Token bucket rate limiter (for global limits)
    public boolean tryAcquireGlobal(String resource, int tokensPerSecond) {
        String key = "bucket:" + resource;
        
        String script = """
            local key = KEYS[1]
            local rate = tonumber(ARGV[1])
            local capacity = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])
            local requested = tonumber(ARGV[4])
            
            local fill_time = capacity / rate
            local ttl = math.floor(fill_time * 2)
            
            local last_tokens = tonumber(redis.call("hget", key, "tokens"))
            if last_tokens == nil then
                last_tokens = capacity
            end
            
            local last_refreshed = tonumber(redis.call("hget", key, "refreshed"))
            if last_refreshed == nil then
                last_refreshed = 0
            end
            
            local delta = math.max(0, now - last_refreshed)
            local filled_tokens = math.min(capacity, last_tokens + (delta * rate / 1000))
            local allowed = filled_tokens >= requested
            local new_tokens = filled_tokens
            
            if allowed then
                new_tokens = filled_tokens - requested
            end
            
            redis.call("hset", key, "tokens", new_tokens)
            redis.call("hset", key, "refreshed", now)
            redis.call("expire", key, ttl)
            
            return allowed and 1 or 0
            """;
        
        Long result = redis.execute(
            new DefaultRedisScript<>(script, Long.class),
            List.of(key),
            String.valueOf(tokensPerSecond),
            String.valueOf(tokensPerSecond * 10), // Burst capacity
            String.valueOf(System.currentTimeMillis()),
            "1"
        );
        
        return result == 1;
    }
}
```

### Deduplication

```java
@Service
public class DeduplicationService {
    
    private final RedisTemplate<String, String> redis;
    
    // Idempotency key check
    public boolean checkAndSetIdempotencyKey(String key) {
        String redisKey = "idem:" + key;
        Boolean isNew = redis.opsForValue().setIfAbsent(
            redisKey, 
            "1", 
            Duration.ofHours(24)
        );
        return isNew != null && isNew;
    }
    
    // Content-based deduplication
    public boolean isDuplicate(String userId, String templateId, String contentHash) {
        String key = "dedup:" + userId + ":" + templateId + ":" + contentHash;
        Boolean isNew = redis.opsForValue().setIfAbsent(
            key, 
            "1", 
            Duration.ofHours(1)
        );
        return isNew == null || !isNew;
    }
    
    // Collapse key for push notifications
    public void setCollapseKey(String userId, String collapseKey, String notificationId) {
        String key = "collapse:" + userId + ":" + collapseKey;
        redis.opsForValue().set(key, notificationId, Duration.ofHours(24));
    }
    
    public String getCollapseKey(String userId, String collapseKey) {
        String key = "collapse:" + userId + ":" + collapseKey;
        return redis.opsForValue().get(key);
    }
}
```

### Cache Stampede Prevention

**What is Cache Stampede?**

Cache stampede (also called "thundering herd") occurs when cached device tokens, user preferences, or templates expire and many requests simultaneously try to regenerate them, overwhelming the database.

**Scenario:**
```
T+0s:   Popular user's device tokens cache expires
T+0s:   10,000 requests arrive simultaneously
T+0s:   All 10,000 requests see cache MISS
T+0s:   All 10,000 requests hit database simultaneously
T+1s:   Database overwhelmed, latency spikes
```

**Prevention Strategy: Distributed Lock + Double-Check Pattern**

This system uses distributed locking to ensure only one request regenerates the cache:

1. **Distributed locking**: Only one thread fetches from database on cache miss
2. **Double-check pattern**: Re-check cache after acquiring lock
3. **TTL jitter**: Add random jitter to TTL to prevent simultaneous expiration

**Implementation:**

```java
@Service
public class DeviceTokenCacheService {
    
    private final RedisTemplate<String, List<DeviceToken>> redis;
    private final DistributedLock lockService;
    private final DeviceTokenRepository deviceTokenRepository;
    
    public List<DeviceToken> getCachedTokens(String userId) {
        String key = "tokens:" + userId;
        
        // 1. Try cache first
        List<DeviceToken> cached = redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:tokens:" + userId;
        boolean acquired = lockService.tryLock(lockKey, Duration.ofSeconds(5));
        
        if (!acquired) {
            // Another thread is fetching, wait briefly
            try {
                Thread.sleep(50);
                cached = redis.opsForValue().get(key);
                if (cached != null) {
                    return cached;
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            // Fall back to database (acceptable degradation)
        }
        
        try {
            // 3. Double-check cache (might have been populated while acquiring lock)
            cached = redis.opsForValue().get(key);
            if (cached != null) {
                return cached;
            }
            
            // 4. Fetch from database (only one thread does this)
            List<DeviceToken> tokens = deviceTokenRepository.findByUserId(userId);
            
            // 5. Populate cache with TTL jitter (add ±10% random jitter to 1h TTL)
            Duration baseTtl = Duration.ofHours(1);
            long jitter = (long)(baseTtl.getSeconds() * 0.1 * (Math.random() * 2 - 1));
            Duration ttl = baseTtl.plusSeconds(jitter);
            
            redis.opsForValue().set(key, tokens, ttl);
            
            return tokens;
            
        } finally {
            if (acquired) {
                lockService.unlock(lockKey);
            }
        }
    }
}
```

**TTL Jitter Benefits:**

- Prevents simultaneous expiration of related cache entries
- Reduces cache stampede probability
- Smooths out cache refresh load over time

---

## 3. Push Notification Deep Dive

### APNS Connection Management

```java
@Service
public class ApnsService {
    
    private final ApnsClient apnsClient;
    private final MeterRegistry metrics;
    
    @PostConstruct
    public void init() throws Exception {
        // Create client with connection pool
        apnsClient = new ApnsClientBuilder()
            .setApnsServer(ApnsClientBuilder.PRODUCTION_APNS_HOST)
            .setSigningKey(ApnsSigningKey.loadFromPkcs8File(
                new File(keyPath),
                teamId,
                keyId
            ))
            .setConcurrentConnections(100) // Connection pool
            .setMetricsListener(new ApnsMetricsListener())
            .build();
    }
    
    public CompletableFuture<PushResult> send(PushNotification notification) {
        ApnsPayloadBuilder payload = new ApnsPayloadBuilder()
            .setAlertTitle(notification.getTitle())
            .setAlertBody(notification.getBody())
            .setSound(notification.getSound())
            .setBadgeNumber(notification.getBadge())
            .setMutableContent(notification.isMutableContent())
            .setContentAvailable(notification.isContentAvailable());
        
        // Add custom data
        notification.getData().forEach(payload::addCustomProperty);
        
        SimpleApnsPushNotification apns = new SimpleApnsPushNotification(
            notification.getDeviceToken(),
            notification.getTopic(),
            payload.build(),
            Instant.now().plus(Duration.ofDays(1)), // Expiration
            DeliveryPriority.IMMEDIATE,
            notification.getCollapseId()
        );
        
        return apnsClient.sendNotification(apns)
            .thenApply(response -> {
                if (response.isAccepted()) {
                    metrics.counter("apns.sent.success").increment();
                    return PushResult.success(response.getApnsId().toString());
                } else {
                    metrics.counter("apns.sent.rejected", 
                        "reason", response.getRejectionReason()).increment();
                    return handleRejection(notification, response);
                }
            })
            .exceptionally(ex -> {
                metrics.counter("apns.sent.error").increment();
                return PushResult.error(ex.getMessage());
            });
    }
    
    private PushResult handleRejection(PushNotification notification, 
                                        PushNotificationResponse<?> response) {
        String reason = response.getRejectionReason();
        
        switch (reason) {
            case "BadDeviceToken":
            case "Unregistered":
                // Invalid token - mark for removal
                deviceTokenService.invalidate(notification.getDeviceToken());
                return PushResult.invalidToken();
                
            case "DeviceTokenNotForTopic":
                // Wrong bundle ID
                return PushResult.configError(reason);
                
            case "TooManyRequests":
                // Rate limited - retry
                return PushResult.retryable(reason);
                
            default:
                return PushResult.error(reason);
        }
    }
}
```

### FCM Batch Sending

```java
@Service
public class FcmService {
    
    private final FirebaseMessaging fcm;
    
    // Send batch of up to 500 messages
    public List<SendResponse> sendBatch(List<PushNotification> notifications) {
        List<Message> messages = notifications.stream()
            .map(this::buildMessage)
            .collect(Collectors.toList());
        
        try {
            BatchResponse response = fcm.sendAll(messages);
            
            List<SendResponse> results = new ArrayList<>();
            for (int i = 0; i < response.getResponses().size(); i++) {
                SendResponse sendResponse = response.getResponses().get(i);
                PushNotification notification = notifications.get(i);
                
                if (sendResponse.isSuccessful()) {
                    results.add(SendResponse.success(
                        sendResponse.getMessageId()
                    ));
                } else {
                    results.add(handleFcmError(
                        notification,
                        sendResponse.getException()
                    ));
                }
            }
            
            return results;
        } catch (FirebaseMessagingException e) {
            // Batch failed entirely
            throw new PushDeliveryException("FCM batch failed", e);
        }
    }
    
    private Message buildMessage(PushNotification notification) {
        Notification.Builder notifBuilder = Notification.builder()
            .setTitle(notification.getTitle())
            .setBody(notification.getBody());
        
        if (notification.getImageUrl() != null) {
            notifBuilder.setImage(notification.getImageUrl());
        }
        
        AndroidConfig android = AndroidConfig.builder()
            .setPriority(mapPriority(notification.getPriority()))
            .setTtl(Duration.ofHours(24).toMillis())
            .setCollapseKey(notification.getCollapseKey())
            .setNotification(AndroidNotification.builder()
                .setIcon(notification.getIcon())
                .setColor(notification.getColor())
                .setSound(notification.getSound())
                .build())
            .build();
        
        return Message.builder()
            .setToken(notification.getDeviceToken())
            .setNotification(notifBuilder.build())
            .setAndroidConfig(android)
            .putAllData(notification.getData())
            .build();
    }
    
    private SendResponse handleFcmError(PushNotification notification, 
                                         FirebaseMessagingException e) {
        String errorCode = e.getMessagingErrorCode().name();
        
        switch (e.getMessagingErrorCode()) {
            case UNREGISTERED:
            case INVALID_ARGUMENT:
                // Invalid token
                deviceTokenService.invalidate(notification.getDeviceToken());
                return SendResponse.invalidToken();
                
            case QUOTA_EXCEEDED:
            case UNAVAILABLE:
                // Retryable
                return SendResponse.retryable(errorCode);
                
            default:
                return SendResponse.error(errorCode);
        }
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Push Notification Technology-Level

**Normal Flow: APNS Push Notification**

```
Step 1: Push Worker Receives Notification
┌─────────────────────────────────────────────────────────────┐
│ Kafka Consumer: Consumes from push-queue-high               │
│ Notification: {                                              │
│   device_token: "abc123...",                                 │
│   title: "Order Confirmed",                                  │
│   body: "Your order #456 has been confirmed",                │
│   data: { order_id: "ORD-456" },                            │
│   priority: "high"                                          │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Build APNS Payload
┌─────────────────────────────────────────────────────────────┐
│ APNS Payload: {                                              │
│   "aps": {                                                   │
│     "alert": {                                               │
│       "title": "Order Confirmed",                            │
│       "body": "Your order #456 has been confirmed"          │
│     },                                                       │
│     "sound": "default",                                      │
│     "badge": 1,                                              │
│     "mutable-content": 1                                     │
│   },                                                         │
│   "order_id": "ORD-456"                                      │
│ }                                                            │
│                                                              │
│ Payload size: ~200 bytes                                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Send to APNS
┌─────────────────────────────────────────────────────────────┐
│ APNS Connection:                                             │
│ - Host: api.push.apple.com                                   │
│ - Port: 443 (HTTPS)                                          │
│ - Connection: Persistent (HTTP/2)                            │
│ - Authentication: JWT token (valid 1 hour)                  │
│                                                              │
│ Request:                                                     │
│ POST /3/device/abc123...                                     │
│ Headers:                                                     │
│   apns-topic: com.example.app                                │
│   apns-priority: 10 (immediate)                             │
│   apns-expiration: 86400 (24 hours)                         │
│ Body: JSON payload                                           │
│                                                              │
│ Latency: ~50ms (APNS response)                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: APNS Response
┌─────────────────────────────────────────────────────────────┐
│ Response: 200 OK                                             │
│ Headers:                                                     │
│   apns-id: xyz789...                                         │
│                                                              │
│ Success: Notification accepted by APNS                       │
│ - Will be delivered to device                               │
│ - Delivery confirmation via feedback service                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Device Receives Notification
┌─────────────────────────────────────────────────────────────┐
│ iOS Device:                                                  │
│ - APNS delivers to device                                    │
│ - Device displays notification                              │
│ - User sees: "Order Confirmed"                              │
│ - User taps → App opens with order_id                       │
│                                                              │
│ Total time: ~100-200ms from send to delivery                │
└─────────────────────────────────────────────────────────────┘
```

**Normal Flow: FCM Push Notification**

```
Step 1: Push Worker Receives Notification
┌─────────────────────────────────────────────────────────────┐
│ Kafka Consumer: Consumes from push-queue-normal             │
│ Notification: {                                              │
│   device_token: "fcm_token_xyz...",                          │
│   title: "New Message",                                      │
│   body: "You have a new message from John",                  │
│   data: { message_id: "msg_123" },                           │
│   priority: "normal"                                        │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Build FCM Message
┌─────────────────────────────────────────────────────────────┐
│ FCM Message: {                                               │
│   "token": "fcm_token_xyz...",                               │
│   "notification": {                                          │
│     "title": "New Message",                                  │
│     "body": "You have a new message from John"             │
│   },                                                         │
│   "data": {                                                  │
│     "message_id": "msg_123"                                 │
│   },                                                         │
│   "android": {                                               │
│     "priority": "normal",                                    │
│     "ttl": "86400s"                                          │
│   }                                                          │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Send to FCM
┌─────────────────────────────────────────────────────────────┐
│ FCM API:                                                     │
│ - Endpoint: https://fcm.googleapis.com/v1/projects/...      │
│ - Method: POST                                               │
│ - Authentication: OAuth 2.0 token                           │
│ - Request: Send message                                      │
│                                                              │
│ Latency: ~100ms (FCM response)                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: FCM Response
┌─────────────────────────────────────────────────────────────┐
│ Response: 200 OK                                             │
│ Body: {                                                      │
│   "name": "projects/.../messages/0:123..."                   │
│ }                                                            │
│                                                              │
│ Success: Message ID returned                                 │
│ - FCM will deliver to device                                │
│ - Delivery status via FCM API                                │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Invalid Device Token**

```
Step 1: Push Sent to Invalid Token
┌─────────────────────────────────────────────────────────────┐
│ APNS Request: POST /3/device/invalid_token_xyz               │
│ Payload: { ... }                                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: APNS Rejects
┌─────────────────────────────────────────────────────────────┐
│ Response: 400 Bad Request                                    │
│ Headers:                                                     │
│   apns-id: xyz789...                                         │
│ Body: {                                                      │
│   "reason": "BadDeviceToken"                                 │
│ }                                                            │
│                                                              │
│ Error: Device token is invalid or unregistered              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Application Handling
┌─────────────────────────────────────────────────────────────┐
│ Application:                                                 │
│ 1. Mark device token as invalid                             │
│ 2. Remove token from user's device list                     │
│ 3. Update notification status: FAILED                       │
│ 4. Log for analytics                                         │
│ 5. Do NOT retry (permanent failure)                         │
│                                                              │
│ Future notifications: Skip this device                      │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: APNS Rate Limiting**

```
Step 1: High Volume Push Requests
┌─────────────────────────────────────────────────────────────┐
│ Scenario: 100,000 push notifications/second                  │
│ APNS rate limit: 10,000/second per connection               │
│ Current connections: 10                                      │
│ Total capacity: 100,000/second                              │
│                                                              │
│ Spike: 150,000 requests/second                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: APNS Rate Limits
┌─────────────────────────────────────────────────────────────┐
│ Response: 429 Too Many Requests                             │
│ Headers:                                                     │
│   retry-after: 60                                            │
│ Body: {                                                      │
│   "reason": "TooManyRequests"                               │
│ }                                                            │
│                                                              │
│ Impact: 50,000 requests/second rejected                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Application Handling
┌─────────────────────────────────────────────────────────────┐
│ Application:                                                 │
│ 1. Back off for 60 seconds                                  │
│ 2. Queue rejected notifications for retry                   │
│ 3. Scale up APNS connections (if possible)                  │
│ 4. Throttle producer (reduce send rate)                     │
│ 5. Retry after retry-after period                           │
│                                                              │
│ Recovery:                                                    │
│ - Retry queued notifications                                │
│ - Gradually increase send rate                               │
│ - Monitor for rate limit errors                             │
└─────────────────────────────────────────────────────────────┘
```

**Retry Logic:**

```
Push Notification Retry Strategy:
1. First retry: 1 second (transient network error)
2. Second retry: 5 seconds (APNS temporary failure)
3. Third retry: 30 seconds (rate limit)
4. Max retries: 3
5. After max retries: Move to DLQ

Retry Conditions:
- Network timeout → Retry
- 429 Too Many Requests → Retry with backoff
- 500 Internal Server Error → Retry
- 400 Bad Request → No retry (invalid token)
- 410 Unregistered → No retry (remove token)
```

---

## 4. Email Delivery Deep Dive

### SendGrid Integration

```java
@Service
public class SendGridEmailService {
    
    private final SendGrid sendGrid;
    private final EmailTemplateRenderer templateRenderer;
    
    public EmailResult send(EmailNotification notification) {
        try {
            // Render template
            String htmlContent = templateRenderer.render(
                notification.getTemplateId(),
                notification.getData()
            );
            
            // Build email
            Email from = new Email(
                notification.getFromEmail(),
                notification.getFromName()
            );
            Email to = new Email(notification.getToEmail());
            
            Mail mail = new Mail();
            mail.setFrom(from);
            mail.setSubject(notification.getSubject());
            
            Personalization personalization = new Personalization();
            personalization.addTo(to);
            
            // Add merge tags
            notification.getData().forEach((key, value) -> 
                personalization.addDynamicTemplateData(key, value)
            );
            
            mail.addPersonalization(personalization);
            mail.addContent(new Content("text/html", htmlContent));
            
            // Tracking settings
            TrackingSettings tracking = new TrackingSettings();
            tracking.setClickTracking(new ClickTracking(true, true));
            tracking.setOpenTracking(new OpenTracking(true));
            mail.setTrackingSettings(tracking);
            
            // Custom headers for tracking
            mail.addHeader("X-Notification-ID", notification.getId());
            
            // Categories for analytics
            mail.addCategory(notification.getCategory());
            
            // Send
            Request request = new Request();
            request.setMethod(Method.POST);
            request.setEndpoint("mail/send");
            request.setBody(mail.build());
            
            Response response = sendGrid.api(request);
            
            if (response.getStatusCode() == 202) {
                String messageId = response.getHeaders().get("X-Message-Id");
                return EmailResult.success(messageId);
            } else {
                return EmailResult.error(
                    response.getStatusCode(),
                    response.getBody()
                );
            }
        } catch (IOException e) {
            return EmailResult.error(e.getMessage());
        }
    }
    
    // Handle webhook events from SendGrid
    public void handleWebhook(List<SendGridEvent> events) {
        for (SendGridEvent event : events) {
            String notificationId = event.getHeader("X-Notification-ID");
            
            switch (event.getEvent()) {
                case "delivered":
                    updateStatus(notificationId, DeliveryStatus.DELIVERED);
                    break;
                case "open":
                    recordOpen(notificationId, event.getTimestamp());
                    break;
                case "click":
                    recordClick(notificationId, event.getUrl(), event.getTimestamp());
                    break;
                case "bounce":
                    handleBounce(notificationId, event);
                    break;
                case "dropped":
                    handleDrop(notificationId, event.getReason());
                    break;
                case "spamreport":
                    handleSpamReport(notificationId, event);
                    break;
            }
        }
    }
    
    private void handleBounce(String notificationId, SendGridEvent event) {
        String bounceType = event.getBounceType();
        String email = event.getEmail();
        
        if ("hard".equals(bounceType)) {
            // Permanent failure - suppress email
            suppressionService.addToSuppressionList(email, "hard_bounce");
            updateStatus(notificationId, DeliveryStatus.BOUNCED);
        } else {
            // Soft bounce - may retry
            updateStatus(notificationId, DeliveryStatus.SOFT_BOUNCE);
        }
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Email Delivery Technology-Level

**Normal Flow: SendGrid Email Delivery**

```
Step 1: Email Worker Receives Notification
┌─────────────────────────────────────────────────────────────┐
│ Kafka Consumer: Consumes from email-queue-high               │
│ Notification: {                                              │
│   to: "user@example.com",                                    │
│   template_id: "order_confirmation",                         │
│   subject: "Order Confirmed",                                │
│   data: { order_id: "ORD-456", total: "$99.99" },          │
│   priority: "high"                                           │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Render Email Template
┌─────────────────────────────────────────────────────────────┐
│ Template Engine:                                             │
│ - Load template: order_confirmation.html                     │
│ - Replace variables:                                         │
│   {{order_id}} → "ORD-456"                                   │
│   {{total}} → "$99.99"                                       │
│ - Generate HTML: ~5KB                                        │
│                                                              │
│ Latency: ~10ms                                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Build SendGrid Request
┌─────────────────────────────────────────────────────────────┐
│ SendGrid Mail Object:                                        │
│ {                                                            │
│   "from": { "email": "noreply@example.com" },               │
│   "to": [{ "email": "user@example.com" }],                  │
│   "subject": "Order Confirmed",                              │
│   "content": [{                                              │
│     "type": "text/html",                                     │
│     "value": "<html>...</html>"                              │
│   }],                                                        │
│   "tracking_settings": {                                     │
│     "click_tracking": { "enable": true },                    │
│     "open_tracking": { "enable": true }                      │
│   },                                                         │
│   "custom_args": {                                           │
│     "notification_id": "notif_abc123"                        │
│   }                                                          │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Send to SendGrid API
┌─────────────────────────────────────────────────────────────┐
│ SendGrid API:                                                │
│ - Endpoint: https://api.sendgrid.com/v3/mail/send            │
│ - Method: POST                                               │
│ - Authentication: Bearer token (API key)                    │
│ - Request: Mail object                                        │
│                                                              │
│ Latency: ~200ms (SendGrid response)                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: SendGrid Response
┌─────────────────────────────────────────────────────────────┐
│ Response: 202 Accepted                                       │
│ Headers:                                                     │
│   X-Message-Id: abc123...                                    │
│                                                              │
│ Success: Email accepted for delivery                         │
│ - SendGrid will deliver to recipient                         │
│ - Delivery events via webhook                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 6: Email Delivery (Async)
┌─────────────────────────────────────────────────────────────┐
│ SendGrid → Recipient's Mail Server:                          │
│ - SMTP connection established                                │
│ - Email transferred                                          │
│ - Delivery confirmation                                      │
│                                                              │
│ Recipient's Mail Server → User's Inbox:                      │
│ - Email stored in inbox                                      │
│ - User receives email                                        │
│                                                              │
│ Total time: ~1-5 seconds from send to inbox                 │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Hard Bounce (Invalid Email)**

```
Step 1: Email Sent to Invalid Address
┌─────────────────────────────────────────────────────────────┐
│ SendGrid Request:                                            │
│ POST /v3/mail/send                                           │
│ To: "invalid@nonexistent-domain.com"                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: SendGrid Attempts Delivery
┌─────────────────────────────────────────────────────────────┐
│ SendGrid:                                                    │
│ - DNS lookup for nonexistent-domain.com                      │
│ - DNS resolution fails                                       │
│ - Delivery attempt fails                                     │
│                                                              │
│ Response: 202 Accepted (queued for retry)                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: SendGrid Retries and Bounces
┌─────────────────────────────────────────────────────────────┐
│ SendGrid:                                                    │
│ - Retries delivery (3 attempts)                             │
│ - All attempts fail                                          │
│ - Generates bounce event                                     │
│                                                              │
│ Webhook Event:                                               │
│ {                                                            │
│   "event": "bounce",                                         │
│   "email": "invalid@nonexistent-domain.com",                │
│   "type": "hard",                                            │
│   "reason": "550 5.1.1 User unknown"                        │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Application Handling
┌─────────────────────────────────────────────────────────────┐
│ Application Webhook Handler:                                 │
│ 1. Receive bounce event                                      │
│ 2. Check bounce type: "hard" (permanent)                     │
│ 3. Add email to suppression list                             │
│ 4. Update notification status: BOUNCED                      │
│ 5. Mark user email as invalid                                │
│ 6. Future emails to this address: Suppressed                │
│                                                              │
│ Suppression List:                                            │
│ - Prevents future sends to invalid addresses                │
│ - Reduces bounce rate                                        │
│ - Protects sender reputation                                 │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Soft Bounce (Temporary Failure)**

```
Step 1: Email Sent to Valid Address
┌─────────────────────────────────────────────────────────────┐
│ SendGrid Request:                                            │
│ POST /v3/mail/send                                           │
│ To: "user@example.com" (valid address)                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Recipient Mail Server Temporarily Unavailable
┌─────────────────────────────────────────────────────────────┐
│ SendGrid → Recipient Mail Server:                            │
│ - SMTP connection attempt                                    │
│ - Connection timeout (server overloaded)                    │
│ - Delivery fails                                             │
│                                                              │
│ Response: 202 Accepted (queued for retry)                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: SendGrid Retries
┌─────────────────────────────────────────────────────────────┐
│ SendGrid Retry Schedule:                                     │
│ - Retry 1: 5 minutes (server may be back up)                 │
│ - Retry 2: 15 minutes                                        │
│ - Retry 3: 30 minutes                                       │
│ - Retry 4: 1 hour                                            │
│                                                              │
│ Retry 2: Success                                             │
│ - Server recovered                                           │
│ - Email delivered                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Delivery Confirmation
┌─────────────────────────────────────────────────────────────┐
│ Webhook Event:                                               │
│ {                                                            │
│   "event": "delivered",                                      │
│   "email": "user@example.com",                               │
│   "timestamp": 1705312800                                    │
│ }                                                            │
│                                                              │
│ Application:                                                 │
│ - Update notification status: DELIVERED                     │
│ - Record delivery time                                       │
│ - No suppression (soft bounce recovered)                   │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: SendGrid Rate Limiting**

```
Step 1: High Volume Email Sends
┌─────────────────────────────────────────────────────────────┐
│ Scenario: 200,000 emails/second                              │
│ SendGrid rate limit: 100,000/second                          │
│ Current plan: Pro (100K/sec limit)                          │
│                                                              │
│ Spike: 200,000 requests/second                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: SendGrid Rate Limits
┌─────────────────────────────────────────────────────────────┐
│ Response: 429 Too Many Requests                             │
│ Body: {                                                      │
│   "errors": [{                                               │
│     "message": "Rate limit exceeded",                       │
│     "field": null,                                           │
│     "help": "https://sendgrid.com/docs/..."                 │
│   }]                                                         │
│ }                                                            │
│                                                              │
│ Impact: 100,000 requests/second rejected                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Application Handling
┌─────────────────────────────────────────────────────────────┐
│ Application:                                                 │
│ 1. Throttle producer (reduce send rate to 100K/sec)        │
│ 2. Queue rejected emails for retry                          │
│ 3. Implement exponential backoff                            │
│ 4. Scale up SendGrid plan (if needed)                       │
│ 5. Retry queued emails gradually                            │
│                                                              │
│ Recovery:                                                    │
│ - Gradually increase send rate                              │
│ - Monitor for rate limit errors                             │
│ - Consider upgrading SendGrid plan                           │
└─────────────────────────────────────────────────────────────┘
```

**Retry Logic:**

```
Email Delivery Retry Strategy:
1. First retry: 5 minutes (soft bounce)
2. Second retry: 15 minutes
3. Third retry: 30 minutes
4. Fourth retry: 1 hour
5. Max retries: 4 (SendGrid handles)
6. After max retries: Hard bounce → Suppress

Retry Conditions:
- Soft bounce → Retry (up to 4 times)
- Hard bounce → No retry (suppress)
- Rate limit → Retry with backoff
- Network timeout → Retry
- 500 Internal Server Error → Retry
```

**Bounce Handling:**

```
Bounce Types:
1. Hard Bounce (Permanent):
   - Invalid email address
   - Domain doesn't exist
   - Mailbox full (permanent)
   - Action: Suppress immediately

2. Soft Bounce (Temporary):
   - Mailbox full (temporary)
   - Server temporarily unavailable
   - Message too large
   - Action: Retry (up to 4 times)

3. Spam Report:
   - User marked as spam
   - Action: Suppress immediately
```

---

## 5. Step-by-Step Simulations

### Simulation 1: High-Priority Order Notification

```
Scenario: Order confirmed, send push + email immediately

Step 1: API receives request (T=0ms)
├── POST /v1/notifications
├── user_id: user_123
├── template_id: order_confirmation
├── channels: [push, email]
├── priority: high
└── data: { order_id: "ORD-456", total: "$99.99" }

Step 2: Notification Service processes (T=5ms)
├── Validate template exists ✓
├── Load user preferences from Redis
├── Check rate limit (100/hour) ✓
├── Check deduplication ✓
├── Get device tokens (2 devices)
├── Get email address
└── Response: 202 Accepted, notification_id: notif_abc123

Step 3: Enqueue to Kafka (T=10ms)
├── push-queue-high: 2 messages (one per device)
├── email-queue-high: 1 message
└── Key: user_123 (for ordering)

Step 4: Push Worker consumes (T=50ms)
├── Device 1 (iOS): Send to APNS
├── Device 2 (Android): Send to FCM
├── Both succeed
└── Publish delivery events

Step 5: Email Worker consumes (T=100ms)
├── Render HTML template
├── Send via SendGrid
├── Receive 202 Accepted
└── Publish sent event

Step 6: Delivery confirmation (T=200ms)
├── APNS confirms delivery
├── FCM confirms delivery
├── Update notification status: delivered
└── Trigger webhook to client

Total time: ~200ms from request to delivery
```

### Simulation 2: Bulk Marketing Campaign

```
Scenario: Send promotional notification to 5M users

Step 1: API receives campaign request (T=0)
├── POST /v1/notifications/segment
├── segment: { country: "US", subscription: "premium" }
├── template_id: black_friday_promo
├── channels: [push, email]
├── priority: low
└── throttle: 50,000/second

Step 2: Segment resolution (T=1s)
├── Query user service for segment
├── Estimated recipients: 5,000,000
├── Create campaign record
└── Response: 202 Accepted, campaign_id: camp_xyz

Step 3: Fan-out to users (T=1s - 100s)
├── Batch users in groups of 10,000
├── For each batch:
│   ├── Load user preferences
│   ├── Filter opted-out users
│   ├── Enqueue notifications
│   └── Throttle at 50K/sec
├── Total enqueued: 4,500,000 (500K opted out)
└── Fan-out time: ~90 seconds

Step 4: Push delivery (T=100s - 200s)
├── 200 push workers consuming
├── Throughput: 500,000/sec
├── Total push: 4,000,000 (some email-only)
├── Delivery rate: 95%
└── Time: ~8 seconds

Step 5: Email delivery (T=100s - 300s)
├── 200 email workers consuming
├── Throughput: 100,000/sec
├── Total emails: 4,500,000
├── SendGrid rate limit: 100K/sec
└── Time: ~45 seconds

Step 6: Analytics (T=300s+)
├── Delivery events aggregated
├── Open tracking begins
├── Click tracking begins
└── Dashboard updated

Campaign Summary:
├── Total targeted: 5,000,000
├── Delivered (push): 3,800,000 (95%)
├── Delivered (email): 4,410,000 (98%)
├── Total time: ~5 minutes
└── Cost: Push $0, Email $441
```

