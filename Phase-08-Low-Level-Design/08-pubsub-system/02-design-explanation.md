# 📬 In-Memory Pub/Sub System - Design Explanation

## STEP 2: Detailed Design Explanation

This document covers the design decisions, SOLID principles application, design patterns used, and complexity analysis for the Pub/Sub System.

---

## STEP 3: SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `Message` | Hold message data | Message format changes |
| `Topic` | Manage subscribers for one topic | Topic behavior changes |
| `Subscription` | Link subscriber to topic with filter | Subscription features change |
| `Subscriber` | Define message handling contract | Handler interface changes |
| `PubSubService` | Coordinate topics and delivery | Service-level changes |
| `MessageFilters` | Provide common filter implementations | New filter types |

**SRP in Action:**

```java
// Message ONLY holds data
public class Message {
    private final String id;
    private final Object payload;
    private final Map<String, String> headers;
    // No behavior beyond getters
}

// Topic ONLY manages its subscribers
public class Topic {
    public void addSubscriber(Subscription sub) { }
    public void removeSubscriber(Subscription sub) { }
    public int publish(Message msg) { }
}

// PubSubService coordinates but doesn't implement low-level logic
public class PubSubService {
    public void publish(Message msg) {
        Topic topic = topics.get(msg.getTopic());
        topic.publish(msg);  // Delegates to Topic
    }
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Subscriber Types:**

```java
// No changes to existing code needed!

// Add batching subscriber
public class BatchSubscriber implements Subscriber {
    private final List<Message> batch = new ArrayList<>();
    private final int batchSize;
    private final Consumer<List<Message>> processor;
    
    @Override
    public void onMessage(Message message) {
        batch.add(message);
        if (batch.size() >= batchSize) {
            processor.accept(new ArrayList<>(batch));
            batch.clear();
        }
    }
}

// Add retry subscriber
public class RetrySubscriber implements Subscriber {
    private final Subscriber delegate;
    private final int maxRetries;
    
    @Override
    public void onMessage(Message message) {
        for (int i = 0; i <= maxRetries; i++) {
            try {
                delegate.onMessage(message);
                return;
            } catch (Exception e) {
                if (i == maxRetries) throw e;
            }
        }
    }
}
```

**Adding New Delivery Modes:**

```java
// Add priority-based delivery
public class PriorityPubSubService extends PubSubService {
    private final PriorityQueue<Message> priorityQueue;
    
    @Override
    public void publish(Message message) {
        int priority = Integer.parseInt(
            message.getHeader("priority", "5"));
        priorityQueue.offer(new PrioritizedMessage(message, priority));
        processQueue();
    }
}
```

---

### 3. Liskov Substitution Principle (LSP)

**All Subscribers are interchangeable:**

```java
// Any Subscriber implementation works
public void testSubscriber(Subscriber subscriber) {
    Message msg = Message.of("test", "Hello");
    subscriber.onMessage(msg);  // Works for any implementation
}

testSubscriber(new PrintSubscriber("test"));
testSubscriber(new QueueSubscriber("test"));
testSubscriber(new CallbackSubscriber("test", m -> {}));
```

**LSP Contract:**
- `onMessage()` must handle any valid Message
- `getId()` must return a non-null identifier
- `onError()` should not throw exceptions

---

### 4. Interface Segregation Principle (ISP)

**Current Design:**

```java
// Minimal Subscriber interface
public interface Subscriber {
    void onMessage(Message message);
    String getId();
    default void onError(Message message, Throwable error) { }
}
```

**If we needed more features:**

```java
// Separate interfaces
public interface MessageHandler {
    void onMessage(Message message);
}

public interface ErrorHandler {
    void onError(Message message, Throwable error);
}

public interface Identifiable {
    String getId();
}

public interface AcknowledgingSubscriber extends MessageHandler {
    void acknowledge(Message message);
    void reject(Message message);
}

// Subscribers implement only what they need
public class SimpleSubscriber implements MessageHandler, Identifiable {
    // No error handling required
}
```

---

### 5. Dependency Inversion Principle (DIP)

**Current Implementation:**

```java
// PubSubService depends on concrete Topic
public class PubSubService {
    private final Map<String, Topic> topics;  // Concrete
}
```

**Better with DIP:**

```java
// Define abstractions
public interface TopicManager {
    void addSubscription(Subscription sub);
    void removeSubscription(Subscription sub);
    int publish(Message message);
}

public interface MessageDelivery {
    void deliver(Subscription sub, Message message);
}

// PubSubService depends on abstractions
public class PubSubService {
    private final Map<String, TopicManager> topics;
    private final MessageDelivery delivery;
    
    public PubSubService(MessageDelivery delivery) {
        this.delivery = delivery;
    }
}

// Different delivery implementations
public class SyncDelivery implements MessageDelivery { }
public class AsyncDelivery implements MessageDelivery { }
public class BatchDelivery implements MessageDelivery { }
```

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. PubSubService coordinates, Topic manages subscriptions, Subscription represents subscriber, Message carries data. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new subscription types, delivery strategies) without modifying existing code. Observer pattern and interfaces enable this. | N/A | - |
| **LSP** | PASS | All Subscriber implementations properly implement the Subscriber interface contract. All Topic implementations are substitutable. | N/A | - |
| **ISP** | PASS | Subscriber interface is minimal and focused. Clients only implement onMessage. Topic interface is well-segregated. | N/A | - |
| **DIP** | WEAK | PubSubService depends on concrete Topic. Could depend on TopicManager and MessageDelivery interfaces. Mentioned in DIP section but not fully implemented. | Extract TopicManager and MessageDelivery interfaces, use dependency injection | More abstraction layers, but improves testability and delivery strategy flexibility |

---

## Design Patterns Used

### 1. Observer Pattern

**Where:** Core pub/sub mechanism

```java
// Subject (Observable)
public class Topic {
    private Set<Subscription> subscriptions;  // Observers
    
    public void publish(Message message) {
        for (Subscription sub : subscriptions) {
            sub.deliver(message);  // Notify observers
        }
    }
}

// Observer
public interface Subscriber {
    void onMessage(Message message);  // Update method
}
```

**Benefits:**
- Loose coupling between publishers and subscribers
- Dynamic subscription management
- One-to-many notification

---

### 2. Builder Pattern

**Where:** Message construction

```java
Message message = Message.builder()
    .topic("orders")
    .payload(orderData)
    .header("priority", "HIGH")
    .header("source", "web")
    .timestamp(Instant.now())
    .build();
```

**Benefits:**
- Clear, readable message construction
- Optional fields without constructor overloading
- Immutable messages

---

### 3. Strategy Pattern

**Where:** Message filtering

```java
// Strategy interface
public interface Predicate<Message> {
    boolean test(Message message);
}

// Concrete strategies
Predicate<Message> byHeader = MessageFilters.hasHeader("type", "order");
Predicate<Message> byPayload = MessageFilters.payloadContains("urgent");
Predicate<Message> combined = MessageFilters.and(byHeader, byPayload);

// Context uses strategy
Subscription sub = new Subscription(topic, subscriber, combined);
```

---

### 4. Factory Pattern

**Where:** MessageFilters

```java
public class MessageFilters {
    public static Predicate<Message> hasHeader(String key, String value) {
        return msg -> value.equals(msg.getHeader(key));
    }
    
    public static Predicate<Message> payloadType(Class<?> type) {
        return msg -> type.isInstance(msg.getPayload());
    }
}
```

---

## Why Alternatives Were Rejected

### Alternative 1: Direct Method Calls

```java
// Rejected
public class OrderService {
    private NotificationService notificationService;
    private InventoryService inventoryService;
    private AnalyticsService analyticsService;
    
    public void processOrder(Order order) {
        // Direct coupling to all services
        notificationService.notify(order);
        inventoryService.update(order);
        analyticsService.track(order);
    }
}
```

**Why rejected:**
- Tight coupling
- Hard to add/remove services
- OrderService knows about all consumers

### Alternative 2: Callback Registration

```java
// Rejected
public class EventEmitter {
    private List<Consumer<Object>> callbacks = new ArrayList<>();
    
    public void on(Consumer<Object> callback) {
        callbacks.add(callback);
    }
    
    public void emit(Object event) {
        callbacks.forEach(cb -> cb.accept(event));
    }
}
```

**Why rejected:**
- No topic separation
- No filtering capability
- Type safety issues

### Alternative 3: Shared Queue

```java
// Rejected
public class SharedQueue {
    private BlockingQueue<Message> queue = new LinkedBlockingQueue<>();
    
    public void publish(Message msg) {
        queue.offer(msg);
    }
    
    public Message consume() {
        return queue.poll();
    }
}
```

**Why rejected:**
- Only one consumer gets each message
- No fan-out capability
- Consumers compete for messages

---

## Thread Safety Analysis

### CopyOnWriteArraySet for Subscriptions

```java
private final Set<Subscription> subscriptions = new CopyOnWriteArraySet<>();
```

**Why CopyOnWriteArraySet?**
- Read-heavy workload (publishing iterates, subscription changes are rare)
- No ConcurrentModificationException during iteration
- Safe for concurrent publish and subscribe

**Trade-offs:**
- Write operations copy entire set (expensive for frequent changes)
- Good for small to medium subscriber counts
- For many subscribers, consider ReadWriteLock

### ConcurrentHashMap for Topics

```java
private final Map<String, Topic> topics = new ConcurrentHashMap<>();
```

**Why ConcurrentHashMap?**
- Thread-safe without global lock
- Good for concurrent topic access
- computeIfAbsent is atomic

### Async Delivery Thread Safety

```java
private void publishAsync(Topic topic, Message message) {
    for (Subscription subscription : topic.getSubscriptions()) {
        executor.submit(() -> {
            subscription.deliver(message);  // Each in separate thread
        });
    }
}
```

**Considerations:**
- Each subscriber handles messages in different threads
- Subscribers must be thread-safe
- Message ordering not guaranteed per subscriber

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you implement message replay?

**Answer:**

```java
public class ReplayableTopic extends Topic {
    private final Deque<Message> history;
    private final int maxHistory;
    
    public void replay(Subscription subscription, int count) {
        List<Message> toReplay = getRecentMessages(count);
        for (Message msg : toReplay) {
            subscription.deliver(msg);
        }
    }
    
    public void replayFrom(Subscription subscription, Instant from) {
        synchronized (history) {
            for (Message msg : history) {
                if (msg.getTimestamp().isAfter(from)) {
                    subscription.deliver(msg);
                }
            }
        }
    }
}
```

---

### Q2: How would you implement message deduplication?

**Answer:**

```java
public class DeduplicatingSubscriber implements Subscriber {
    private final Subscriber delegate;
    private final Set<String> seenIds;
    private final int maxSize;
    
    @Override
    public void onMessage(Message message) {
        if (seenIds.add(message.getId())) {
            delegate.onMessage(message);
            
            // Evict old IDs if too many
            if (seenIds.size() > maxSize) {
                // Remove oldest (would need LinkedHashSet or LRU cache)
                seenIds.remove(seenIds.iterator().next());
            }
        }
    }
}
```

---

### Q3: How would you implement backpressure?

**Answer:**

```java
public class BackpressureSubscriber implements Subscriber {
    private final Subscriber delegate;
    private final Semaphore permits;
    private final BlockingQueue<Message> buffer;
    
    public BackpressureSubscriber(Subscriber delegate, int maxInFlight) {
        this.delegate = delegate;
        this.permits = new Semaphore(maxInFlight);
        this.buffer = new LinkedBlockingQueue<>();
        startProcessor();
    }
    
    @Override
    public void onMessage(Message message) {
        if (!permits.tryAcquire()) {
            // Buffer or drop
            buffer.offer(message);
            return;
        }
        
        try {
            delegate.onMessage(message);
        } finally {
            permits.release();
        }
    }
    
    private void startProcessor() {
        new Thread(() -> {
            while (true) {
                try {
                    Message msg = buffer.take();
                    permits.acquire();
                    delegate.onMessage(msg);
                    permits.release();
                } catch (InterruptedException e) {
                    break;
                }
            }
        }).start();
    }
}
```

---

### Q4: What would you do differently with more time?

**Answer:**

1. **Add message persistence** - Store messages to disk/database for durability
2. **Add consumer groups** - Load balance across multiple consumers of same subscription
3. **Add message TTL** - Auto-expire old messages based on time-to-live
4. **Add metrics** - Track publish/subscribe rates, latencies, queue depths
5. **Add wildcards** - Subscribe to topic patterns (e.g., "orders.*", "user.#.events")
6. **Add transactions** - Publish multiple messages atomically
7. **Add message ordering** - Guarantee ordered delivery within partitions
8. **Add dead letter queue** - Handle failed message deliveries

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `createTopic` | O(1) | HashMap put |
| `subscribe` | O(1) | Set add |
| `unsubscribe` | O(n) | Set remove (CopyOnWrite) |
| `publish` (sync) | O(s) | s = subscribers |
| `publish` (async) | O(s) | Submit tasks |

### Space Complexity

| Component | Space |
|-----------|-------|
| Topics map | O(t) | t = number of topics |
| Per topic | O(s) | s = subscriptions |
| Message history | O(h) | h = history size |
| Per message | O(m) | m = payload + headers size |

### Bottlenecks at Scale

**10x Usage (100 → 1K topics, 1K → 10K subscribers):**
- Problem: Topic lookup becomes noticeable, message delivery to large subscriber lists becomes expensive, memory for message history grows
- Solution: Index topics for faster lookup, batch message delivery, implement message retention policies
- Tradeoff: More memory for indexes, batching adds latency

**100x Usage (100 → 10K topics, 1K → 100K subscribers):**
- Problem: Single instance can't handle all topics/subscribers, message delivery bottlenecks, memory exhausted
- Solution: Partition topics across multiple broker instances, use distributed message queue (Kafka, RabbitMQ), implement message persistence
- Tradeoff: Distributed system complexity, need partition coordination and message replication


### Q1: How would you implement message acknowledgment?

```java
public interface AckSubscriber extends Subscriber {
    void onMessage(Message message, AckHandle handle);
}

public class AckHandle {
    private final Message message;
    private final Topic topic;
    private volatile boolean acknowledged;
    
    public void ack() {
        acknowledged = true;
    }
    
    public void nack() {
        // Redeliver
        topic.publish(message);
    }
}

public class AckTopic extends Topic {
    private final Map<String, Message> pendingAcks = new ConcurrentHashMap<>();
    private final ScheduledExecutorService scheduler;
    
    @Override
    public int publish(Message message) {
        pendingAcks.put(message.getId(), message);
        
        // Schedule redelivery if not acked
        scheduler.schedule(() -> {
            if (pendingAcks.containsKey(message.getId())) {
                redeliver(message);
            }
        }, 30, TimeUnit.SECONDS);
        
        return super.publish(message);
    }
}
```

### Q2: How would you implement message ordering?

```java
public class OrderedTopic extends Topic {
    private final Map<String, Long> subscriberSequences = new ConcurrentHashMap<>();
    
    @Override
    public int publish(Message message) {
        // Add sequence number
        long seq = sequenceGenerator.getAndIncrement();
        Message sequencedMsg = Message.builder()
            .from(message)
            .header("_seq", String.valueOf(seq))
            .build();
        
        return super.publish(sequencedMsg);
    }
}

public class OrderedSubscriber implements Subscriber {
    private final PriorityQueue<Message> buffer = new PriorityQueue<>(
        Comparator.comparingLong(m -> Long.parseLong(m.getHeader("_seq"))));
    private long expectedSeq = 0;
    
    @Override
    public void onMessage(Message message) {
        buffer.offer(message);
        
        // Process in order
        while (!buffer.isEmpty() && 
               Long.parseLong(buffer.peek().getHeader("_seq")) == expectedSeq) {
            processInOrder(buffer.poll());
            expectedSeq++;
        }
    }
}
```

### Q3: How would you implement dead letter queue?

```java
public class DLQPubSubService extends PubSubService {
    private final Topic deadLetterTopic;
    private final int maxRetries;
    private final Map<String, Integer> retryCount = new ConcurrentHashMap<>();
    
    @Override
    public void publish(Message message) {
        try {
            super.publish(message);
        } catch (Exception e) {
            handleFailure(message, e);
        }
    }
    
    private void handleFailure(Message message, Exception e) {
        int retries = retryCount.merge(message.getId(), 1, Integer::sum);
        
        if (retries >= maxRetries) {
            // Move to DLQ
            Message dlqMessage = Message.builder()
                .from(message)
                .header("_error", e.getMessage())
                .header("_originalTopic", message.getTopic())
                .topic("dead-letter")
                .build();
            deadLetterTopic.publish(dlqMessage);
            retryCount.remove(message.getId());
        } else {
            // Retry with backoff
            scheduler.schedule(() -> publish(message), 
                retries * 1000L, TimeUnit.MILLISECONDS);
        }
    }
}
```

### Q4: How would you scale to multiple servers?

```java
// Use Redis Pub/Sub for cross-server communication
public class DistributedPubSubService extends PubSubService {
    private final JedisPool jedisPool;
    
    @Override
    public void publish(Message message) {
        // Publish locally
        super.publish(message);
        
        // Publish to Redis for other servers
        try (Jedis jedis = jedisPool.getResource()) {
            jedis.publish(message.getTopic(), serialize(message));
        }
    }
    
    public void startRedisSubscriber() {
        new Thread(() -> {
            try (Jedis jedis = jedisPool.getResource()) {
                jedis.psubscribe(new JedisPubSub() {
                    @Override
                    public void onMessage(String channel, String message) {
                        // Deliver to local subscribers
                        Message msg = deserialize(message);
                        Topic topic = getTopic(channel);
                        if (topic != null) {
                            topic.publish(msg);
                        }
                    }
                }, "*");
            }
        }).start();
    }
}
```

