# Chat System - Production Deep Dives (Core)

## Overview

This document covers the technical implementation details for key components: WebSocket management, message ordering, delivery guarantees, presence tracking, typing indicators, and group chat optimization.

---

## 1. WebSocket Connection Management

### Connection Handler

```java
@Component
public class ChatWebSocketHandler extends TextWebSocketHandler {

    private final SessionRegistry sessionRegistry;
    private final MessageRouter messageRouter;
    private final PresenceService presenceService;

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        String userId = extractUserId(session);
        String deviceId = extractDeviceId(session);

        // Register session
        sessionRegistry.register(userId, deviceId, session);

        // Update presence
        presenceService.setOnline(userId, deviceId);

        // Send pending messages
        deliverPendingMessages(userId, deviceId, session);

        log.info("User {} connected from device {}", userId, deviceId);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        String userId = extractUserId(session);

        try {
            ChatMessage chatMessage = parseMessage(message.getPayload());

            switch (chatMessage.getType()) {
                case "message":
                    handleChatMessage(userId, chatMessage);
                    break;
                case "ack":
                    handleAcknowledgment(userId, chatMessage);
                    break;
                case "typing":
                    handleTypingIndicator(userId, chatMessage);
                    break;
                case "ping":
                    handlePing(session);
                    break;
            }
        } catch (Exception e) {
            log.error("Error handling message from {}", userId, e);
            sendError(session, "Invalid message format");
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        String userId = extractUserId(session);
        String deviceId = extractDeviceId(session);

        // Unregister session
        sessionRegistry.unregister(userId, deviceId);

        // Update presence (check if other devices still connected)
        presenceService.handleDisconnect(userId, deviceId);

        log.info("User {} disconnected from device {}", userId, deviceId);
    }

    private void deliverPendingMessages(String userId, String deviceId, WebSocketSession session) {
        List<Message> pending = messageQueue.getPending(userId, deviceId);

        for (Message msg : pending) {
            try {
                session.sendMessage(new TextMessage(serialize(msg)));
            } catch (IOException e) {
                log.error("Failed to deliver pending message", e);
                break;  // Will retry on next connection
            }
        }
    }
}
```

### Session Registry (Redis-backed)

```java
@Service
public class SessionRegistry {

    private final RedisTemplate<String, String> redis;
    private final Map<String, WebSocketSession> localSessions = new ConcurrentHashMap<>();

    public void register(String userId, String deviceId, WebSocketSession session) {
        String sessionKey = "session:" + userId + ":" + deviceId;
        String serverId = getServerId();

        // Store locally
        localSessions.put(sessionKey, session);

        // Store in Redis for cross-server routing
        redis.opsForHash().putAll(sessionKey, Map.of(
            "server_id", serverId,
            "connected_at", Instant.now().toString()
        ));
        redis.expire(sessionKey, Duration.ofMinutes(5));

        // Track user's devices
        redis.opsForSet().add("devices:" + userId, deviceId);
    }

    public void unregister(String userId, String deviceId) {
        String sessionKey = "session:" + userId + ":" + deviceId;

        localSessions.remove(sessionKey);
        redis.delete(sessionKey);
        redis.opsForSet().remove("devices:" + userId, deviceId);
    }

    public Optional<String> findServer(String userId, String deviceId) {
        String sessionKey = "session:" + userId + ":" + deviceId;
        return Optional.ofNullable(
            (String) redis.opsForHash().get(sessionKey, "server_id")
        );
    }

    public Set<String> getActiveDevices(String userId) {
        return redis.opsForSet().members("devices:" + userId);
    }

    // Heartbeat to keep session alive
    public void refreshSession(String userId, String deviceId) {
        String sessionKey = "session:" + userId + ":" + deviceId;
        redis.expire(sessionKey, Duration.ofMinutes(5));
    }
}
```

---

## 1.5. WebSocket Technology-Level Simulation

### A) CONCEPT: What is WebSocket?

WebSocket is a full-duplex communication protocol that enables persistent, bidirectional connections between client and server. Unlike HTTP (request-response), WebSocket allows the server to push messages to clients without waiting for requests.

**What problems does WebSocket solve here?**

1. **Real-time delivery**: Messages arrive instantly without polling
2. **Low latency**: No HTTP overhead per message
3. **Efficient**: Single connection handles all messages
4. **Bidirectional**: Client and server can send anytime

### B) OUR USAGE: How We Use WebSocket Here

**Connection Lifecycle:**

```
1. Client initiates: ws://chat.example.com/ws?token=abc123
2. Server validates token, establishes connection
3. Client sends heartbeat (ping) every 30 seconds
4. Server sends messages as they arrive
5. Connection closes on logout or timeout
```

**Connection Management:**

- Max connections per server: 50,000
- Heartbeat interval: 30 seconds
- Timeout: 60 seconds (2 missed heartbeats)
- Reconnection: Exponential backoff (1s, 2s, 4s, 8s, max 30s)

### C) REAL STEP-BY-STEP SIMULATION

**Normal Flow: Message Delivery via WebSocket**

```
Step 1: User Sends Message
┌─────────────────────────────────────────────────────────────┐
│ Client: User types "Hello" and sends                        │
│ WebSocket: Send {                                            │
│   "type": "message",                                        │
│   "conversation_id": "conv_123",                            │
│   "text": "Hello",                                          │
│   "timestamp": 1705312800000                                │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Server Receives and Processes
┌─────────────────────────────────────────────────────────────┐
│ WebSocket Handler:                                           │
│ 1. Validate message format                                  │
│ 2. Store in Cassandra (durability)                          │
│ 3. Generate sequence number: 1001                          │
│ 4. Update message with sequence                             │
│ 5. Find recipient's active sessions                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Real-time Delivery to Recipient
┌─────────────────────────────────────────────────────────────┐
│ Session Registry:                                            │
│ - Recipient: user_456                                        │
│ - Active devices: [device_phone, device_laptop]            │
│ - Both connected to same server                             │
│                                                              │
│ WebSocket Delivery:                                          │
│ - Send to device_phone: {message with seq 1001}           │
│ - Send to device_laptop: {message with seq 1001}          │
│ - Both receive in < 10ms                                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Cross-Server Delivery (if needed)
┌─────────────────────────────────────────────────────────────┐
│ If recipient on different server:                           │
│ 1. Publish to Redis pub/sub: "user:user_456:message"       │
│ 2. Other servers subscribed receive event                   │
│ 3. Target server delivers via its WebSocket connection     │
│ 4. Total latency: < 50ms                                    │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: WebSocket Connection Drops**

```
Scenario: Network interruption during message delivery

┌─────────────────────────────────────────────────────────────┐
│ T+0s: User sends message "Hello"                            │
│ T+5ms: Server receives, stores in DB                        │
│ T+10ms: Server attempts WebSocket delivery                 │
│ T+11ms: Network interruption (WiFi drops)                   │
│ T+11ms: WebSocket connection closes                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+12ms: Server detects connection closed                    │
│ T+13ms: Message queued for offline delivery                │
│ T+14ms: Push notification sent (if mobile)                 │
│ T+15ms: Message stored in pending queue                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+30s: Client reconnects (exponential backoff)             │
│ T+30s: Client sends sync request: "last_seq: 1000"         │
│ T+31ms: Server sends missed messages: [1001, 1002, ...]    │
│ T+32ms: Client receives "Hello" message                    │
│ T+33ms: Message displayed to user                          │
└─────────────────────────────────────────────────────────────┘
```

**Idempotency Handling:**

```
Problem: Message delivered twice (network retry + reconnection)

Solution: Sequence numbers + client deduplication
1. Each message has unique sequence number
2. Client tracks received sequence numbers
3. Duplicate messages (same sequence) ignored
4. Server idempotent: Same message_id stored once

Example:
- Message sent twice → Same sequence number → Client deduplicates
- Reconnection sync → Only sends messages after last_seq
```

**Traffic Spike Handling:**

```
Scenario: Celebrity sends message to group with 1M members

┌─────────────────────────────────────────────────────────────┐
│ Normal: 1-on-1 message (2 WebSocket sends)                 │
│ Spike: Group message to 1M members                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Handling Strategy:                                           │
│ 1. Store message once in Cassandra                          │
│ 2. Fan-out on read (not on write)                           │
│ 3. Members pull when they open group                        │
│ 4. Push notification sent to all members                    │
│ 5. WebSocket delivery only to active users                  │
│                                                              │
│ Result:                                                      │
│ - Message stored: 1 write                                   │
│ - WebSocket sends: ~100K (active users only)               │
│ - Push notifications: 1M (all members)                      │
│ - No WebSocket overload                                     │
└─────────────────────────────────────────────────────────────┘
```

**Hot Connection Mitigation:**

```
Problem: Popular user receives messages from millions

Mitigation:
1. Message batching: Batch multiple messages per WebSocket frame
2. Read receipts: Async, batched (not per message)
3. Presence updates: Throttled (max 1 per second)
4. Connection pooling: Multiple WebSocket servers

If hot connection occurs:
- Monitor connection message rate
- Implement message queuing on client
- Consider separate connection for high-volume chats
```

**Recovery Behavior:**

```
Auto-healing:
- Connection drop → Client reconnects → Sync missed messages → Automatic
- Server crash → Load balancer routes to healthy server → Automatic
- Network partition → Client detects → Reconnect → Automatic

Human intervention:
- Persistent connection failures → Check network/firewall
- Server overload → Scale WebSocket servers
- Message delivery stuck → Investigate queue/DB issues
```

---

## 1.6. Redis for Session Management - Technology-Level Simulation

### A) CONCEPT: What is Redis?

Redis is an in-memory data structure store. For Chat System, Redis stores session information, presence status, and message queues to enable cross-server communication and fast lookups.

**What problems does Redis solve here?**

1. **Cross-server routing**: Find which server a user is connected to
2. **Presence tracking**: Real-time online/offline status
3. **Message queuing**: Offline message delivery
4. **Low latency**: Sub-millisecond lookups

### B) OUR USAGE: How We Use Redis Here

**Session Storage (Hash):**

```
Key: session:{user_id}:{device_id}
Value: Hash {
  "server_id": "server_123",
  "connected_at": "2024-01-20T10:00:00Z"
}
TTL: 5 minutes (refreshed on heartbeat)
```

**Device Tracking (Set):**

```
Key: devices:{user_id}
Value: Set of device IDs ["phone", "laptop", "tablet"]
TTL: None (manually managed)
```

**Presence Status (String):**

```
Key: presence:{user_id}
Value: "online" or "offline"
TTL: 5 minutes (refreshed on activity)
```

### C) REAL STEP-BY-STEP SIMULATION

**Normal Flow: User Connects and Sends Message**

```
Step 1: User Connects
┌─────────────────────────────────────────────────────────────┐
│ Client: WebSocket connection established                    │
│ Server: Register session                                     │
│ Redis: HSET session:user_123:phone {                       │
│   "server_id": "server_5",                                  │
│   "connected_at": "2024-01-20T10:00:00Z"                   │
│ }                                                            │
│ Redis: SADD devices:user_123 phone                          │
│ Redis: SET presence:user_123 "online" EX 300               │
│ Latency: ~2ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: User Sends Message
┌─────────────────────────────────────────────────────────────┐
│ Message: "Hello" to user_456                                │
│ Redis: SMEMBERS devices:user_456                            │
│ Returns: ["phone", "laptop"]                                │
│ Redis: HGET session:user_456:phone server_id               │
│ Returns: "server_8"                                         │
│ Redis: HGET session:user_456:laptop server_id              │
│ Returns: "server_5" (local server)                         │
│ Latency: ~3ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Message Routing
┌─────────────────────────────────────────────────────────────┐
│ Server 5:                                                    │
│ - Delivers to laptop locally (WebSocket)                   │
│ - Publishes to Redis pub/sub for server_8                  │
│ Server 8:                                                    │
│ - Receives pub/sub event                                    │
│ - Delivers to phone (WebSocket)                            │
│ Total latency: < 50ms                                       │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Redis Node Failure**

```
Scenario: Redis node handling session data crashes

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Redis node 2 crashes (handles slots 5461-10922)      │
│ T+0-5s: Session lookups fail for affected users            │
│   - HGET session:user_123:phone → Error                    │
│   - SMEMBERS devices:user_123 → Error                       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+5s: Redis cluster detects failure                          │
│ T+10s: Replica promoted to primary                          │
│ T+15s: Cluster topology updated                              │
│ T+20s: All requests succeeding again                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Impact:                                                      │
│ - Session lookups fail for ~1/3 of users (15-20 seconds)  │
│ - Circuit breaker: Fallback to local session cache         │
│ - Message routing: Uses local cache (may be stale)         │
│ - Users reconnect: New sessions registered                  │
│ - No message loss (messages queued in Cassandra)            │
└─────────────────────────────────────────────────────────────┘
```

**Idempotency Handling:**

```
Problem: Session registered twice (network retry)

Solution: Redis operations are idempotent
1. HSET: Overwrites existing hash (idempotent)
2. SADD: Adds to set (idempotent if already member)
3. SET: Overwrites existing value (idempotent)

Example:
- Session registered twice → Same Redis key → Overwrites (idempotent)
- Device added twice → SADD → No duplicate (idempotent)
```

**Traffic Spike Handling:**

```
Scenario: Millions of users reconnect after outage

┌─────────────────────────────────────────────────────────────┐
│ Normal: 1,000 connections/second                            │
│ Spike: 100,000 connections/second (reconnection storm)    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Redis Handling:                                             │
│ - HSET operations: O(1) per operation                      │
│ - Pipeline operations: Batch 100 operations                 │
│ - Connection pooling: Reuse connections                     │
│ - Read replicas: Distribute read load                       │
│ - Result: 100K ops/sec handled by Redis cluster            │
│                                                              │
│ Mitigation:                                                 │
│ - Exponential backoff on client reconnection               │
│ - Rate limiting per user (max 10 reconnects/minute)        │
│ - Circuit breaker: Fallback to local cache if Redis slow   │
└─────────────────────────────────────────────────────────────┘
```

**Hot Key Mitigation:**

```
Problem: Popular user's session accessed by millions

Mitigation:
1. Local session cache: Cache hottest sessions in app memory
2. Read replicas: Distribute reads across replicas
3. Connection pooling: Reuse Redis connections
4. Pipeline operations: Batch multiple operations

If hot key occurs:
- Monitor key access patterns
- Add read replicas
- Consider sharding by user_id (if needed)
```

**Recovery Behavior:**

```
Auto-healing:
- Redis node failure → Cluster promotes replica → Automatic
- Network partition → Read from replica → Automatic
- Session lookup failure → Fallback to local cache → Automatic

Human intervention:
- Cluster-wide failure → Manual failover to DR region
- Persistent hot keys → Scale Redis or optimize access pattern
- Session data corruption → Clear and rebuild from Cassandra
```

### Hot Partition Mitigation (Kafka Message Queue)

**Problem:** If a popular conversation receives many messages, one Kafka partition could become hot, creating a bottleneck for message processing.

**Example Scenario:**
```
Normal conversation: 10 messages/hour → Partition 5 (hash(conversation_id) % partitions)
Popular group chat: 1000 messages/hour → Partition 5 (same partition!)
Result: Partition 5 receives 100x traffic, becomes bottleneck
Message processing workers for partition 5 become overwhelmed
```

**Detection:**
- Monitor partition lag per partition (Kafka metrics)
- Alert when partition lag > 2x average
- Track partition throughput metrics (messages/second per partition)
- Monitor consumer lag per partition
- Alert threshold: Lag > 2x average for 5 minutes

**Mitigation Strategies:**

1. **Partition Key Strategy:**
   - Partition by `conversation_id` hash (distributes across conversations)
   - For very active conversations, consider composite key (conversation_id + timestamp_bucket)
   - This distributes high-volume conversations across multiple partitions

2. **Partition Rebalancing:**
   - If hot partition detected, increase partition count
   - Rebalance existing partitions (Kafka supports this)
   - Distribute hot conversation messages across multiple partitions

3. **Consumer Scaling:**
   - Add more consumers for hot partitions
   - Auto-scale consumers based on partition lag
   - Each consumer handles subset of hot partition

4. **Rate Limiting:**
   - Rate limit per conversation (prevent single conversation from overwhelming)
   - Throttle high-volume conversations
   - Buffer messages locally if rate limit exceeded

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
- Partition throughput (messages/second per partition)
- Consumer lag per partition
- Alert threshold: Lag > 2x average for 5 minutes

### Cache Stampede Prevention (Redis Session Cache)

**What is Cache Stampede?**

Cache stampede occurs when session cache entries expire and many requests simultaneously try to regenerate session data, overwhelming the backend services.

**Scenario:**

```
T+0s:   Popular user's session cache expires
T+0s:   10,000 requests arrive simultaneously
T+0s:   All 10,000 requests see cache MISS
T+0s:   All 10,000 requests hit database/services simultaneously
T+1s:   Backend overwhelmed, latency spikes
```

**Prevention Strategy: Distributed Lock + Double-Check Pattern**

For session-related cache misses (e.g., user profile lookups), this system uses distributed locking:

1. **Distributed locking**: Only one thread fetches from database on cache miss
2. **Double-check pattern**: Re-check cache after acquiring lock
3. **TTL jitter**: Add random jitter to TTL to prevent simultaneous expiration

**Implementation:**

```java
@Service
public class UserProfileCacheService {

    private final RedisTemplate<String, UserProfile> redis;
    private final DistributedLock lockService;
    private final UserRepository userRepository;

    public UserProfile getCachedProfile(String userId) {
        String key = "profile:" + userId;

        // 1. Try cache first
        UserProfile cached = redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }

        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:profile:" + userId;
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
            UserProfile profile = userRepository.findById(userId);

            if (profile != null) {
                // 5. Populate cache with TTL jitter (add ±10% random jitter)
                Duration baseTtl = Duration.ofHours(1);
                long jitter = (long)(baseTtl.getSeconds() * 0.1 * (Math.random() * 2 - 1));
                Duration ttl = baseTtl.plusSeconds(jitter);

                redis.opsForValue().set(key, profile, ttl);
            }

            return profile;

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

## 2. Message Ordering

### Sequence Number Generation

```java
@Service
public class SequenceService {

    private final RedisTemplate<String, Long> redis;

    /**
     * Generate monotonically increasing sequence number per conversation
     */
    public long getNextSequence(String conversationId) {
        String key = "seq:" + conversationId;
        return redis.opsForValue().increment(key);
    }

    /**
     * Batch allocation for high-throughput conversations
     */
    public SequenceRange allocateRange(String conversationId, int count) {
        String key = "seq:" + conversationId;
        long end = redis.opsForValue().increment(key, count);
        long start = end - count + 1;
        return new SequenceRange(start, end);
    }
}
```

### Message Ordering on Client

```java
public class ClientMessageBuffer {

    private final TreeMap<Long, Message> buffer = new TreeMap<>();
    private long lastDeliveredSequence = 0;

    /**
     * Add message to buffer and deliver in-order messages
     */
    public List<Message> addMessage(Message message) {
        buffer.put(message.getSequenceNumber(), message);
        return deliverInOrder();
    }

    private List<Message> deliverInOrder() {
        List<Message> toDeliver = new ArrayList<>();

        while (!buffer.isEmpty()) {
            long nextExpected = lastDeliveredSequence + 1;
            Message next = buffer.get(nextExpected);

            if (next != null) {
                toDeliver.add(next);
                buffer.remove(nextExpected);
                lastDeliveredSequence = nextExpected;
            } else {
                // Gap detected, wait for missing message
                break;
            }
        }

        return toDeliver;
    }

    /**
     * Request missing messages if gap persists
     */
    public List<Long> getMissingSequences() {
        if (buffer.isEmpty()) return Collections.emptyList();

        List<Long> missing = new ArrayList<>();
        long expected = lastDeliveredSequence + 1;
        long highest = buffer.lastKey();

        for (long seq = expected; seq < highest; seq++) {
            if (!buffer.containsKey(seq)) {
                missing.add(seq);
            }
        }

        return missing;
    }
}
```

---

## 3. Message Delivery Guarantees

### At-Least-Once Delivery

```java
@Service
public class MessageDeliveryService {

    private final MessageRepository messageRepository;
    private final SessionRegistry sessionRegistry;
    private final MessageQueue pendingQueue;

    public void deliverMessage(Message message, String recipientId) {
        // 1. Store message first (durability)
        messageRepository.save(message);

        // 2. Try real-time delivery
        Set<String> devices = sessionRegistry.getActiveDevices(recipientId);

        if (devices.isEmpty()) {
            // User offline - queue and send push notification
            queueForOfflineDelivery(message, recipientId);
            sendPushNotification(message, recipientId);
            return;
        }

        // 3. Deliver to all active devices
        for (String deviceId : devices) {
            deliverToDevice(message, recipientId, deviceId);
        }
    }

    private void deliverToDevice(Message message, String userId, String deviceId) {
        Optional<String> serverOpt = sessionRegistry.findServer(userId, deviceId);

        if (serverOpt.isEmpty()) {
            // Session expired, queue message
            pendingQueue.add(userId, deviceId, message);
            return;
        }

        String serverId = serverOpt.get();

        if (isLocalServer(serverId)) {
            // Deliver locally
            WebSocketSession session = sessionRegistry.getLocalSession(userId, deviceId);
            sendToSession(session, message);
        } else {
            // Route to remote server
            routeToServer(serverId, message, userId, deviceId);
        }
    }

    // Retry logic for failed deliveries
    @Scheduled(fixedRate = 5000)
    public void retryFailedDeliveries() {
        List<PendingDelivery> pending = pendingQueue.getPendingOlderThan(Duration.ofSeconds(30));

        for (PendingDelivery delivery : pending) {
            if (delivery.getRetryCount() > 5) {
                // Give up, rely on sync
                pendingQueue.remove(delivery);
                continue;
            }

            try {
                deliverToDevice(delivery.getMessage(),
                               delivery.getUserId(),
                               delivery.getDeviceId());
                pendingQueue.remove(delivery);
            } catch (Exception e) {
                delivery.incrementRetry();
                pendingQueue.update(delivery);
            }
        }
    }
}
```

### Deduplication on Client

```java
public class ClientMessageDeduplicator {

    private final Set<String> seenMessageIds = Collections.newSetFromMap(
        new LinkedHashMap<String, Boolean>(1000, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<String, Boolean> eldest) {
                return size() > 10000;  // Keep last 10K message IDs
            }
        }
    );

    public boolean isDuplicate(String messageId) {
        return !seenMessageIds.add(messageId);
    }

    public void processMessage(Message message) {
        if (isDuplicate(message.getMessageId())) {
            log.debug("Duplicate message ignored: {}", message.getMessageId());
            return;
        }

        // Process message
        displayMessage(message);
        sendAcknowledgment(message.getMessageId());
    }
}
```

---

## 4. Presence Service

### Presence Tracking

```java
@Service
public class PresenceService {

    private final RedisTemplate<String, String> redis;
    private final PresenceBroadcaster broadcaster;

    private static final Duration PRESENCE_TTL = Duration.ofSeconds(60);
    private static final Duration LAST_SEEN_TTL = Duration.ofDays(30);

    public void setOnline(String userId, String deviceId) {
        String presenceKey = "presence:" + userId;
        String previousStatus = redis.opsForValue().get(presenceKey);

        // Set online
        redis.opsForValue().set(presenceKey, "online", PRESENCE_TTL);

        // Clear last_seen (user is online)
        redis.delete("last_seen:" + userId);

        // Broadcast if status changed
        if (!"online".equals(previousStatus)) {
            broadcaster.broadcastPresenceChange(userId, "online");
        }
    }

    public void handleDisconnect(String userId, String deviceId) {
        // Check if user has other active devices
        Set<String> devices = redis.opsForSet().members("devices:" + userId);

        if (devices == null || devices.isEmpty()) {
            // No other devices, set offline
            setOffline(userId);
        }
        // Else: keep online (other devices still connected)
    }

    private void setOffline(String userId) {
        String presenceKey = "presence:" + userId;

        // Set offline
        redis.opsForValue().set(presenceKey, "offline", PRESENCE_TTL);

        // Set last_seen
        redis.opsForValue().set(
            "last_seen:" + userId,
            Instant.now().toString(),
            LAST_SEEN_TTL
        );

        broadcaster.broadcastPresenceChange(userId, "offline");
    }

    public void refreshPresence(String userId) {
        String presenceKey = "presence:" + userId;
        redis.expire(presenceKey, PRESENCE_TTL);
    }

    public PresenceStatus getPresence(String userId) {
        String status = redis.opsForValue().get("presence:" + userId);

        if ("online".equals(status)) {
            return PresenceStatus.online();
        }

        String lastSeen = redis.opsForValue().get("last_seen:" + userId);
        return PresenceStatus.offline(lastSeen != null ? Instant.parse(lastSeen) : null);
    }

    public Map<String, PresenceStatus> getBulkPresence(List<String> userIds) {
        List<String> keys = userIds.stream()
            .map(id -> "presence:" + id)
            .collect(Collectors.toList());

        List<String> statuses = redis.opsForValue().multiGet(keys);

        Map<String, PresenceStatus> result = new HashMap<>();
        for (int i = 0; i < userIds.size(); i++) {
            String userId = userIds.get(i);
            String status = statuses.get(i);
            result.put(userId, "online".equals(status) ?
                PresenceStatus.online() : getPresence(userId));
        }

        return result;
    }
}
```

### Presence Broadcasting

```java
@Service
public class PresenceBroadcaster {

    private final SessionRegistry sessionRegistry;
    private final ContactService contactService;
    private final RedisMessagePublisher redisPublisher;

    public void broadcastPresenceChange(String userId, String status) {
        // Get contacts who should receive this update
        List<String> subscribers = getPresenceSubscribers(userId);

        PresenceUpdate update = PresenceUpdate.builder()
            .userId(userId)
            .status(status)
            .timestamp(Instant.now())
            .build();

        // Publish to Redis for cross-server delivery
        redisPublisher.publish("presence:" + userId, serialize(update));

        // Deliver to local subscribers
        for (String subscriberId : subscribers) {
            Set<String> devices = sessionRegistry.getActiveDevices(subscriberId);
            for (String deviceId : devices) {
                deliverPresenceUpdate(subscriberId, deviceId, update);
            }
        }
    }

    private List<String> getPresenceSubscribers(String userId) {
        // Only notify users who have an active chat with this user
        // Not all contacts (would be too many updates)
        return contactService.getActiveChats(userId);
    }
}
```

---

## 5. Typing Indicators

### Typing Service

```java
@Service
public class TypingService {

    private final RedisTemplate<String, String> redis;
    private final SessionRegistry sessionRegistry;

    private static final Duration TYPING_TTL = Duration.ofSeconds(5);

    public void setTyping(String userId, String conversationId, boolean isTyping) {
        String key = "typing:" + conversationId + ":" + userId;

        if (isTyping) {
            redis.opsForValue().set(key, "1", TYPING_TTL);
        } else {
            redis.delete(key);
        }

        // Broadcast to other participants
        broadcastTyping(userId, conversationId, isTyping);
    }

    private void broadcastTyping(String userId, String conversationId, boolean isTyping) {
        List<String> participants = getConversationParticipants(conversationId);

        TypingIndicator indicator = TypingIndicator.builder()
            .conversationId(conversationId)
            .userId(userId)
            .isTyping(isTyping)
            .build();

        for (String participantId : participants) {
            if (!participantId.equals(userId)) {
                deliverTypingIndicator(participantId, indicator);
            }
        }
    }

    public List<String> getTypingUsers(String conversationId) {
        String pattern = "typing:" + conversationId + ":*";
        Set<String> keys = redis.keys(pattern);

        return keys.stream()
            .map(key -> key.substring(key.lastIndexOf(":") + 1))
            .collect(Collectors.toList());
    }
}
```

---

## 6. Group Chat Optimization

### Group Message Service

```java
@Service
public class GroupMessageService {

    private final MessageRepository messageRepository;
    private final GroupRepository groupRepository;
    private final MessageDeliveryService deliveryService;

    private static final int SMALL_GROUP_THRESHOLD = 100;

    public void sendGroupMessage(Message message, String groupId) {
        // 1. Validate sender is member
        if (!groupRepository.isMember(groupId, message.getSenderId())) {
            throw new UnauthorizedException("Not a group member");
        }

        // 2. Generate sequence number
        long sequence = sequenceService.getNextSequence(groupId);
        message.setSequenceNumber(sequence);

        // 3. Store message once
        messageRepository.save(message);

        // 4. Get group members
        List<String> members = groupRepository.getMembers(groupId);

        // 5. Fan-out strategy based on group size
        if (members.size() <= SMALL_GROUP_THRESHOLD) {
            // Small group: fan-out on write
            fanOutOnWrite(message, members);
        } else {
            // Large group: fan-out on read
            fanOutOnRead(message, groupId);
        }
    }

    private void fanOutOnWrite(Message message, List<String> members) {
        // Parallel delivery to all members
        members.parallelStream()
            .filter(memberId -> !memberId.equals(message.getSenderId()))
            .forEach(memberId -> deliveryService.deliverMessage(message, memberId));
    }

    private void fanOutOnRead(Message message, String groupId) {
        // Store in group's message stream
        // Members will pull when they open the group

        // Only push notification to members
        List<String> members = groupRepository.getMembers(groupId);
        for (String memberId : members) {
            if (!memberId.equals(message.getSenderId())) {
                notificationService.sendGroupNotification(memberId, message);
            }
        }
    }
}
```

### Read Receipts for Groups

```java
@Service
public class GroupReadReceiptService {

    private final RedisTemplate<String, String> redis;

    public void markAsRead(String messageId, String userId) {
        redis.opsForSet().add("read:" + messageId, userId);

        // Notify sender periodically, not per read
        int readCount = redis.opsForSet().size("read:" + messageId).intValue();
        if (shouldNotifySender(readCount)) {
            notifySender(messageId, readCount);
        }
    }

    public ReadReceiptSummary getReadSummary(String messageId, int totalRecipients) {
        Set<String> readers = redis.opsForSet().members("read:" + messageId);
        int readCount = readers != null ? readers.size() : 0;

        return ReadReceiptSummary.builder()
            .messageId(messageId)
            .readCount(readCount)
            .totalRecipients(totalRecipients)
            .readers(readers != null ? new ArrayList<>(readers) : Collections.emptyList())
            .build();
    }

    // UI shows: "Read by 45 of 50"
    // Detail view shows who read
}
```

---

## 7. Database Transactions Deep Dive

### A) CONCEPT: What are Database Transactions in Chat System Context?

Database transactions ensure ACID properties for critical operations like message persistence, conversation creation, and read receipt updates. In a chat system, transactions are essential for maintaining data consistency when multiple operations must succeed or fail together.

**What problems do transactions solve here?**

1. **Atomicity**: Message creation must persist message, update conversation, and update unread counts atomically
2. **Consistency**: Messages can't be lost, read receipts must be accurate
3. **Isolation**: Concurrent message sends don't interfere with each other
4. **Durability**: Once committed, messages persist even if system crashes

**Transaction Isolation Levels:**

| Level           | Description                               | Use Case                            |
| --------------- | ----------------------------------------- | ----------------------------------- |
| READ COMMITTED  | Default, prevents dirty reads             | Reading messages, conversation list |
| REPEATABLE READ | Prevents non-repeatable reads             | Message sequence checks             |
| SERIALIZABLE    | Highest isolation, prevents phantom reads | Critical message operations         |

### B) OUR USAGE: How We Use Transactions Here

**Critical Transactional Operations:**

1. **Message Persistence Transaction**:

   - Insert message into Cassandra
   - Update conversation metadata (last_message_id, last_message_at)
   - Update unread counts for recipients
   - All must succeed or all rollback

2. **Conversation Creation Transaction**:

   - Create conversation record
   - Add participants
   - Create initial message
   - Update user conversation lists

3. **Read Receipt Update Transaction**:
   - Update message read status
   - Update conversation unread count
   - Update user's last_read_message_id

**Transaction Configuration:**

```java
@Transactional(
    isolation = Isolation.REPEATABLE_READ,
    propagation = Propagation.REQUIRED,
    timeout = 10,
    rollbackFor = {Exception.class}
)
public Message sendMessage(SendMessageRequest request) {
    // Transactional operations
}
```

### C) REAL STEP-BY-STEP SIMULATION: Database Transaction Flow

**Normal Flow: Message Persistence Transaction**

```
Step 1: Begin Transaction
┌─────────────────────────────────────────────────────────────┐
│ Cassandra: BEGIN TRANSACTION (Lightweight Transaction)     │
│ Isolation Level: REPEATABLE READ                             │
│ Transaction ID: tx_msg_123                                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Insert Message
┌─────────────────────────────────────────────────────────────┐
│ INSERT INTO messages (message_id, conversation_id,           │
│                       sender_id, content, timestamp, ...)    │
│ VALUES ('msg_456', 'conv_789', 'user_123',                   │
│         'Hello!', 1705312800000, ...)                         │
│                                                              │
│ Partition: conversation_id = 'conv_789'                      │
│ Result: 1 row inserted                                       │
│ Message ID: msg_456                                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Update Conversation Metadata
┌─────────────────────────────────────────────────────────────┐
│ UPDATE conversations                                         │
│ SET last_message_id = 'msg_456',                            │
│     last_message_at = 1705312800000,                        │
│     updated_at = 1705312800000                               │
│ WHERE conversation_id = 'conv_789'                          │
│                                                              │
│ Result: 1 row updated                                       │
│ Conversation metadata updated                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Update Unread Counts
┌─────────────────────────────────────────────────────────────┐
│ UPDATE conversation_participants                             │
│ SET unread_count = unread_count + 1                          │
│ WHERE conversation_id = 'conv_789'                          │
│   AND user_id IN ('user_456', 'user_789')                    │
│   AND user_id != 'user_123'  -- Exclude sender               │
│                                                              │
│ Result: 2 rows updated                                       │
│ Unread counts incremented for recipients                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Commit Transaction
┌─────────────────────────────────────────────────────────────┐
│ Cassandra: COMMIT TRANSACTION                                │
│                                                              │
│ All changes made permanent:                                 │
│ - Message persisted                                         │
│ - Conversation metadata updated                             │
│ - Unread counts updated                                     │
│ - Locks released                                            │
│                                                              │
│ Total time: ~15ms (Cassandra fast writes)                   │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Concurrent Message Sends**

```
Step 1: Two Users Send Messages Simultaneously
┌─────────────────────────────────────────────────────────────┐
│ User A sends message to conversation_123                     │
│ User B sends message to conversation_123                     │
│ Both transactions start simultaneously                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Both Update Conversation Metadata
┌─────────────────────────────────────────────────────────────┐
│ Transaction A: UPDATE conversations                          │
│                SET last_message_id = 'msg_A'                 │
│                WHERE conversation_id = 'conv_123'          │
│                Lock acquired                                │
│                                                              │
│ Transaction B: UPDATE conversations                          │
│                SET last_message_id = 'msg_B'                 │
│                WHERE conversation_id = 'conv_123'           │
│                Waiting for lock (blocked)                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Transaction A Commits
┌─────────────────────────────────────────────────────────────┐
│ Transaction A: COMMIT                                        │
│ - Message A persisted                                       │
│ - Conversation last_message_id = 'msg_A'                   │
│ - Lock released                                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Transaction B Continues
┌─────────────────────────────────────────────────────────────┐
│ Transaction B: Lock acquired                                │
│                UPDATE conversations                          │
│                SET last_message_id = 'msg_B'                 │
│                WHERE conversation_id = 'conv_123'           │
│                                                              │
│ Result: last_message_id = 'msg_B' (overwrites A)            │
│                                                              │
│ Transaction B: COMMIT                                       │
│ - Message B persisted                                       │
│ - Conversation last_message_id = 'msg_B'                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Message Ordering
┌─────────────────────────────────────────────────────────────┐
│ Both messages persisted successfully                         │
│ Message ordering determined by timestamp (not last_message) │
│                                                              │
│ Clients receive messages in timestamp order:                │
│ 1. Message A (timestamp: 1705312800000)                    │
│ 2. Message B (timestamp: 1705312800001)                    │
│                                                              │
│ ✅ Both messages delivered, order preserved                │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Message Persistence Failure**

```
Step 1: Message Insert Fails
┌─────────────────────────────────────────────────────────────┐
│ Transaction: Message persistence in progress                 │
│ - Message insert attempted                                  │
│ - Cassandra node failure detected                           │
│ - Write timeout exception thrown                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Automatic Rollback
┌─────────────────────────────────────────────────────────────┐
│ @Transactional detects exception:                            │
│ - Rolls back entire transaction                              │
│ - No message persisted                                       │
│ - Conversation metadata not updated                         │
│ - Unread counts not updated                                 │
│ - All locks released                                        │
│                                                              │
│ Cassandra: ROLLBACK TRANSACTION                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Application Handling
┌─────────────────────────────────────────────────────────────┐
│ Application catches CassandraWriteTimeoutException:         │
│                                                              │
│ 1. Retry with exponential backoff (up to 3 times)           │
│ 2. If all retries fail:                                     │
│    - Return error to client                                 │
│    - Message queued for retry (Kafka)                        │
│    - Client can retry sending                               │
│ 3. Log failure for monitoring                               │
│                                                              │
│ Message not lost (queued for retry)                        │
└─────────────────────────────────────────────────────────────┘
```

**Isolation Level Behavior: Read Receipt Updates**

```
Scenario: User reads messages while new messages arrive

Transaction A (Read Receipt):          Transaction B (New Message):
─────────────────────────────────────────────────────────────────
BEGIN TRANSACTION                      BEGIN TRANSACTION

SELECT unread_count FROM               INSERT INTO messages (...)
conversation_participants               VALUES ('msg_new', ...)
WHERE conversation_id = 'conv_123'
  AND user_id = 'user_456'             UPDATE conversations
                                        SET last_message_id = 'msg_new'
Result: unread_count = 5

UPDATE conversation_participants        COMMIT
SET unread_count = 0
WHERE conversation_id = 'conv_123'     -- New message added
  AND user_id = 'user_456'

COMMIT

✅ Unread count set to 0               ✅ New message persisted


With REPEATABLE READ:                   With READ COMMITTED:
- Read receipt sees consistent          - Read receipt sees latest
  snapshot (unread_count = 5)            unread_count
- Updates to 0 correctly                - May miss new messages
- New message added after commit         that arrived during
- Unread count becomes 1 (correct)       transaction
```

**Lock Contention: High-Concurrency Message Sends**

```
Step 1: Group Chat - 50 Users Send Messages Simultaneously
┌─────────────────────────────────────────────────────────────┐
│ 50 concurrent transactions trying to send messages          │
│ All to same conversation (group chat)                        │
│                                                              │
│ Lock queue forms on conversation row                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Lock Acquisition Strategy
┌─────────────────────────────────────────────────────────────┐
│ Application: Acquire locks in consistent order              │
│ 1. Lock conversation row (by conversation_id)              │
│ 2. Insert message (partitioned by conversation_id)         │
│ 3. Update unread counts                                     │
│                                                              │
│ Prevents deadlocks by consistent lock ordering             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Optimistic Locking for Unread Counts
┌─────────────────────────────────────────────────────────────┐
│ Instead of row locks, use version-based optimistic locking:│
│                                                              │
│ UPDATE conversation_participants                             │
│ SET unread_count = unread_count + 1,                         │
│     version = version + 1                                    │
│ WHERE conversation_id = 'conv_123'                          │
│   AND version = :expected_version                            │
│                                                              │
│ If version mismatch: Retry with new version                 │
│ Reduces lock contention                                     │
└─────────────────────────────────────────────────────────────┘
```

**Retry Logic for Failed Transactions**

```java
@Retryable(
    value = {CassandraWriteTimeoutException.class,
             DeadlockLoserDataAccessException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2)
)
@Transactional(isolation = Isolation.REPEATABLE_READ)
public Message sendMessage(SendMessageRequest request) {
    // Transaction logic
    // Retries automatically on timeout/deadlock
}
```

**Recovery Behavior:**

```
Transaction Failure → Automatic Retry:
1. Write timeout → Retry with 100ms delay
2. Deadlock detected → Retry with 200ms delay
3. Node failure → Retry on different node
4. Max retries exceeded → Queue message for async processing

Monitoring:
- Track retry rate (alert if > 2%)
- Track write timeout rate (alert if > 1%)
- Track deadlock rate (alert if > 0.5%)
- Track message queue depth (alert if > 1000)
```

---

## 7. Offline Message Handling

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    OFFLINE MESSAGE HANDLING                      │
└─────────────────────────────────────────────────────────────────┘

Sender → Server → Store in DB → Check recipient status
                                        │
                    ┌───────────────────┴───────────────────┐
                    ▼                                       ▼
              ONLINE                                   OFFLINE
                    │                                       │
                    ▼                                       ▼
           Push via WebSocket                    1. Queue in Redis
                                                 2. Send push notification
                                                        │
                                                        ▼
                                                 Recipient comes online
                                                        │
                                                        ▼
                                                 Sync queued messages
```

### Implementation

```java
@Service
public class OfflineMessageService {

    private final MessageRepository messageRepository;
    private final PresenceService presenceService;
    private final MessageQueue messageQueue;
    private final PushNotificationService pushNotificationService;

    public void deliverMessage(Message message, String recipientId) {
        // Always store first
        messageRepository.save(message);

        if (presenceService.isOnline(recipientId)) {
            // Real-time delivery
            pushToUser(recipientId, message);
        } else {
            // Queue for later
            messageQueue.add(recipientId, message);

            // Push notification
            pushNotificationService.send(recipientId,
                "New message from " + message.getSenderName());
        }
    }

    // On reconnect
    public void onUserConnected(String userId) {
        List<Message> pending = messageQueue.getAll(userId);
        for (Message msg : pending) {
            pushToUser(userId, msg);
        }
        messageQueue.clear(userId);
    }
}
```

---

## Summary

| Component          | Technology/Algorithm  | Key Configuration               |
| ------------------ | --------------------- | ------------------------------- |
| WebSocket          | Spring WebSocket      | 100K connections/server         |
| Session store      | Redis                 | 5-min TTL, refresh on heartbeat |
| Message ordering   | Sequence numbers      | Per-conversation counter        |
| Delivery guarantee | At-least-once + dedup | 5 retries, 30s timeout          |
| Presence           | Redis with TTL        | 60s TTL, heartbeat refresh      |
| Typing indicators  | Redis with TTL        | 5s TTL                          |
| Group optimization | Hybrid fan-out        | 100 member threshold            |
