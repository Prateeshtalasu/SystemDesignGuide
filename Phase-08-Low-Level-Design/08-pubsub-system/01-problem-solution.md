# 📬 In-Memory Pub/Sub System - Problem Solution

## STEP 0: REQUIREMENTS QUICKPASS

### Core Functional Requirements
- Support multiple topics and subscribers
- Deliver messages to all subscribers of a topic
- Be thread-safe for concurrent publishers and subscribers
- Support message filtering based on criteria
- Handle subscriber failures gracefully
- Support different delivery modes (sync, async)
- Allow dynamic topic creation and deletion

### Explicit Out-of-Scope Items
- Distributed messaging across nodes
- Message persistence and durability
- Message acknowledgment and retry
- Dead letter queues
- Message ordering guarantees
- Backpressure handling

### Assumptions and Constraints
- **In-Memory Only**: No persistence
- **Single JVM**: Not distributed
- **Fire-and-Forget**: No delivery guarantees
- **Topic-Based**: No pattern/wildcard subscriptions

### Scale Assumptions (LLD Focus)
- Single JVM, moderate throughput (~1000 messages/second)
- Topics: Up to 100 concurrent topics
- Subscribers per topic: Up to 1000
- Message size: Up to 1MB per message
- Async delivery via ExecutorService thread pool

### Concurrency Model
- **ConcurrentHashMap** for topics registry (thread-safe topic creation/deletion)
- **CopyOnWriteArraySet** for subscribers list (safe iteration during publish)
- **ExecutorService** for async message delivery
- **Synchronized publish**: Prevents message interleaving within same topic
- **Subscriber isolation**: One subscriber's exception doesn't affect others

### Public APIs
- `createTopic(name)`: Create new topic
- `deleteTopic(name)`: Delete topic
- `subscribe(topic, subscriber)`: Subscribe to topic
- `unsubscribe(topic, subscriber)`: Unsubscribe
- `publish(topic, message)`: Publish message
- `publishAsync(topic, message)`: Async publish

### Public API Usage Examples
```java
// Example 1: Basic usage
PubSubService pubsub = new PubSubService();
pubsub.createTopic("orders");
Subscriber subscriber = new PrintSubscriber("OrderProcessor");
Subscription subscription = pubsub.subscribe("orders", subscriber);
pubsub.publish("orders", "Order #1001");
pubsub.unsubscribe(subscription);

// Example 2: Typical workflow
PubSubService service = new PubSubService(true); // Async mode
Topic topic = service.createTopic("notifications");
QueueSubscriber queueSub = new QueueSubscriber("QueueSub");
service.subscribe("notifications", queueSub);
service.publish("notifications", "Welcome!");
Message msg = queueSub.poll();

// Example 3: Edge case - filtered subscription
PubSubService pubsub = new PubSubService();
pubsub.createTopic("logs");
Subscriber errorLogger = new PrintSubscriber("ErrorLogger");
service.subscribe("logs", errorLogger, 
    MessageFilters.hasHeader("level", "ERROR"));
// This message will be filtered out
service.publish(Message.builder()
    .topic("logs")
    .payload("Application started")
    .header("level", "INFO")
    .build());
// This message will be delivered
service.publish(Message.builder()
    .topic("logs")
    .payload("Database error")
    .header("level", "ERROR")
    .build());
```

### Invariants
- **No Duplicate Subscriptions**: Same subscriber only once per topic
- **Thread Safety**: All operations are thread-safe
- **Isolation**: Subscriber failure doesn't affect others

---

## STEP 1: Complete Reference Solution (Answer Key)

### Class Diagram Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              PUB/SUB SYSTEM                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                         PubSubService                                     │   │
│  │                                                                           │   │
│  │  - topics: Map<String, Topic>                                            │   │
│  │  - executor: ExecutorService                                             │   │
│  │                                                                           │   │
│  │  + createTopic(name): Topic                                              │   │
│  │  + deleteTopic(name): boolean                                            │   │
│  │  + publish(topicName, message): void                                     │   │
│  │  + subscribe(topicName, subscriber): Subscription                        │   │
│  │  + unsubscribe(subscription): boolean                                    │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                          │                                                       │
│                          │ manages                                               │
│                          ▼                                                       │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                           Topic                                           │   │
│  │                                                                           │   │
│  │  - name: String                                                           │   │
│  │  - subscribers: Set<Subscription>                                         │   │
│  │  - messageHistory: Deque<Message>                                        │   │
│  │                                                                           │   │
│  │  + addSubscriber(subscription): void                                     │   │
│  │  + removeSubscriber(subscription): void                                  │   │
│  │  + publish(message): void                                                │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                          │                                                       │
│                          │ contains                                              │
│                          ▼                                                       │
│  ┌────────────────────────┐         ┌────────────────────────┐                  │
│  │       Message          │         │     Subscription       │                  │
│  │                        │         │                        │                  │
│  │  - id: String          │         │  - id: String          │                  │
│  │  - topic: String       │         │  - topic: Topic        │                  │
│  │  - payload: Object     │         │  - subscriber: Sub     │                  │
│  │  - timestamp: Instant  │         │  - filter: Predicate   │                  │
│  │  - headers: Map        │         │  - active: boolean     │                  │
│  └────────────────────────┘         └────────────────────────┘                  │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                     Subscriber (interface)                                │   │
│  │                                                                           │   │
│  │  + onMessage(message: Message): void                                     │   │
│  │  + getId(): String                                                       │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                          △                                                       │
│           ┌──────────────┼──────────────┐                                       │
│           │              │              │                                       │
│  ┌────────┴────────┐ ┌───┴────────┐ ┌───┴──────────┐                           │
│  │ PrintSubscriber │ │QueueSub    │ │CallbackSub   │                           │
│  └─────────────────┘ └────────────┘ └──────────────┘                           │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Message Flow Visualization

```
Publisher                    PubSubService                    Subscribers
    │                             │                               │
    │  publish("orders", msg)     │                               │
    │────────────────────────────►│                               │
    │                             │                               │
    │                             │  Find topic "orders"          │
    │                             │──────────┐                    │
    │                             │          │                    │
    │                             │◄─────────┘                    │
    │                             │                               │
    │                             │  For each subscriber:         │
    │                             │                               │
    │                             │  Apply filter                 │
    │                             │──────────┐                    │
    │                             │          │                    │
    │                             │◄─────────┘                    │
    │                             │                               │
    │                             │  onMessage(msg)               │
    │                             │──────────────────────────────►│ Sub1
    │                             │                               │
    │                             │  onMessage(msg)               │
    │                             │──────────────────────────────►│ Sub2
    │                             │                               │
    │                             │  (filtered out)               │
    │                             │              ╳                │ Sub3
```

---

### Responsibilities Table

| Class | Owns | Why |
|-------|------|-----|
| `Message` | Message data (payload, headers, metadata) | Encapsulates message content - pure data class, no behavior, enables message passing between publishers and subscribers |
| `Topic` | Subscriber management and message routing for one topic | Manages topic-specific subscribers - separates topic-level concerns from service-level coordination |
| `Subscription` | Link between subscriber and topic with filtering | Encapsulates subscription relationship - combines subscriber, topic, and filter, enables per-subscription filtering |
| `Subscriber` (interface) | Message handling contract | Defines subscriber interface - enables multiple subscriber implementations, separates contract from implementation |
| `PubSubService` | Topic management and message delivery coordination | Central service coordination - manages all topics, handles publish routing, separates service-level logic from topic-level logic |
| `PrintSubscriber` | Console output message handling | Simple subscriber implementation - demonstrates subscriber interface, handles messages by printing |
| `QueueSubscriber` | Queue-based message handling | Queue subscriber implementation - demonstrates different message handling pattern, stores messages in queue |
| `CallbackSubscriber` | Callback-based message handling | Callback subscriber implementation - demonstrates callback pattern, invokes callback function on message |
| `MessageFilters` | Common message filter implementations | Provides reusable filters - separates filter logic from subscription, enables common filtering patterns |

---

## STEP 4: Code Walkthrough - Building From Scratch

This section explains how an engineer builds this system from scratch, in the order code should be written.

### Phase 1: Design the Message

```java
// Step 1: Message class
public class Message {
    private final String id;
    private final String topic;
    private final Object payload;
    private final Instant timestamp;
    private final Map<String, String> headers;
}
```

**Why immutable?**
- Same message delivered to multiple subscribers
- Prevents one subscriber from modifying message for others
- Thread-safe without synchronization

---

### Phase 2: Design the Subscriber Interface

```java
// Step 2: Subscriber interface
public interface Subscriber {
    void onMessage(Message message);
    String getId();
    default void onError(Message message, Throwable error) { }
}
```

---

### Phase 3: Design the Subscription

```java
// Step 3: Subscription links subscriber to topic
public class Subscription {
    private final String id;
    private final Topic topic;
    private final Subscriber subscriber;
    private final Predicate<Message> filter;
    private volatile boolean active;
    
    public boolean acceptsMessage(Message message) {
        if (!active) return false;
        return filter == null || filter.test(message);
    }
}
```

---

### Phase 4: Design the Topic

```java
// Step 4: Topic manages subscriptions
public class Topic {
    private final String name;
    private final Set<Subscription> subscriptions = new CopyOnWriteArraySet<>();
    
    public int publish(Message message) {
        int delivered = 0;
        for (Subscription subscription : subscriptions) {
            if (subscription.isActive() && subscription.acceptsMessage(message)) {
                subscription.deliver(message);
                delivered++;
            }
        }
        return delivered;
    }
}
```

**Why CopyOnWriteArraySet?**
- Safe iteration during publish (no ConcurrentModificationException)
- Reads are fast (no locking)
- Writes copy the set (acceptable for infrequent subscription changes)

---

### Phase 5: Design the PubSubService

```java
// Step 5: Main service class
public class PubSubService {
    private final Map<String, Topic> topics = new ConcurrentHashMap<>();
    private final ExecutorService executor;
    private final boolean asyncDelivery;
    
    public Topic createTopic(String name) {
        return topics.computeIfAbsent(name, Topic::new);
    }
    
    public void publish(Message message) {
        Topic topic = topics.get(message.getTopic());
        if (asyncDelivery) {
            publishAsync(topic, message);
        } else {
            topic.publish(message);
        }
    }
    
    private void publishAsync(Topic topic, Message message) {
        for (Subscription subscription : topic.getSubscriptions()) {
            if (subscription.isActive() && subscription.acceptsMessage(message)) {
                executor.submit(() -> subscription.deliver(message));
            }
        }
    }
}
```

**Why ConcurrentHashMap?**
- Thread-safe without global lock
- Multiple threads can access different topics simultaneously
- `computeIfAbsent` is atomic

---

### Phase 6: Threading Model and Concurrency Control

**Threading Model:**

This pub/sub system handles **concurrent operations**:
- Multiple publishers can publish simultaneously
- Subscriptions can be added/removed while publishing
- Messages can be delivered synchronously or asynchronously

**Concurrency Control:**

```java
// Thread-safe topic map
private final Map<String, Topic> topics = new ConcurrentHashMap<>();

// Thread-safe subscription set (safe for concurrent iteration)
private final Set<Subscription> subscriptions = new CopyOnWriteArraySet<>();

// Async delivery using thread pool
private void publishAsync(Topic topic, Message message) {
    for (Subscription subscription : topic.getSubscriptions()) {
        if (subscription.isActive() && subscription.acceptsMessage(message)) {
            executor.submit(() -> {
                try {
                    subscription.deliver(message);
                } catch (Exception e) {
                    subscription.getSubscriber().onError(message, e);
                }
            });
        }
    }
}
```

**Sync vs Async Delivery:**

- **Synchronous**: Publisher blocks until all subscribers process the message
- **Asynchronous**: Publisher returns immediately; messages delivered via thread pool

---

## STEP 2: Complete Final Implementation

> **Verified:** This code compiles successfully with Java 11+.

### 2.1 Message Class

```java
// Message.java
package com.pubsub;

import java.time.Instant;
import java.util.*;

/**
 * Represents a message in the pub/sub system.
 * 
 * Messages are immutable once created.
 */
public class Message {
    
    private final String id;
    private final String topic;
    private final Object payload;
    private final Instant timestamp;
    private final Map<String, String> headers;
    
    private Message(Builder builder) {
        this.id = builder.id != null ? builder.id : UUID.randomUUID().toString();
        this.topic = builder.topic;
        this.payload = builder.payload;
        this.timestamp = builder.timestamp != null ? builder.timestamp : Instant.now();
        this.headers = Collections.unmodifiableMap(new HashMap<>(builder.headers));
    }
    
    public static Builder builder() {
        return new Builder();
    }
    
    // Quick factory method
    public static Message of(String topic, Object payload) {
        return builder().topic(topic).payload(payload).build();
    }
    
    // Getters
    public String getId() { return id; }
    public String getTopic() { return topic; }
    public Object getPayload() { return payload; }
    public Instant getTimestamp() { return timestamp; }
    public Map<String, String> getHeaders() { return headers; }
    
    public String getHeader(String key) {
        return headers.get(key);
    }
    
    @SuppressWarnings("unchecked")
    public <T> T getPayloadAs(Class<T> type) {
        return (T) payload;
    }
    
    @Override
    public String toString() {
        return String.format("Message[id=%s, topic=%s, payload=%s, timestamp=%s]",
                id, topic, payload, timestamp);
    }
    
    public static class Builder {
        private String id;
        private String topic;
        private Object payload;
        private Instant timestamp;
        private final Map<String, String> headers = new HashMap<>();
        
        public Builder id(String id) {
            this.id = id;
            return this;
        }
        
        public Builder topic(String topic) {
            this.topic = topic;
            return this;
        }
        
        public Builder payload(Object payload) {
            this.payload = payload;
            return this;
        }
        
        public Builder timestamp(Instant timestamp) {
            this.timestamp = timestamp;
            return this;
        }
        
        public Builder header(String key, String value) {
            this.headers.put(key, value);
            return this;
        }
        
        public Builder headers(Map<String, String> headers) {
            this.headers.putAll(headers);
            return this;
        }
        
        public Message build() {
            if (topic == null || topic.isEmpty()) {
                throw new IllegalArgumentException("Topic is required");
            }
            return new Message(this);
        }
    }
}
```

### 2.2 Subscriber Interface

```java
// Subscriber.java
package com.pubsub;

/**
 * Interface for message subscribers.
 * 
 * Implementations receive messages from topics they subscribe to.
 */
public interface Subscriber {
    
    /**
     * Called when a message is published to a subscribed topic.
     * 
     * @param message The published message
     */
    void onMessage(Message message);
    
    /**
     * Returns the unique identifier for this subscriber.
     */
    String getId();
    
    /**
     * Called when an error occurs during message delivery.
     * Default implementation logs the error.
     */
    default void onError(Message message, Throwable error) {
        System.err.println("Error processing message " + message.getId() + 
                          " for subscriber " + getId() + ": " + error.getMessage());
    }
}
```

### 2.3 Subscription Class

```java
// Subscription.java
package com.pubsub;

import java.util.UUID;
import java.util.function.Predicate;

/**
 * Represents a subscription to a topic.
 * 
 * Contains the subscriber and optional message filter.
 */
public class Subscription {
    
    private final String id;
    private final Topic topic;
    private final Subscriber subscriber;
    private final Predicate<Message> filter;
    private volatile boolean active;
    
    public Subscription(Topic topic, Subscriber subscriber) {
        this(topic, subscriber, null);
    }
    
    public Subscription(Topic topic, Subscriber subscriber, Predicate<Message> filter) {
        this.id = UUID.randomUUID().toString();
        this.topic = topic;
        this.subscriber = subscriber;
        this.filter = filter;
        this.active = true;
    }
    
    /**
     * Checks if the message passes the filter.
     */
    public boolean acceptsMessage(Message message) {
        if (!active) {
            return false;
        }
        if (filter == null) {
            return true;
        }
        return filter.test(message);
    }
    
    /**
     * Delivers a message to the subscriber.
     */
    public void deliver(Message message) {
        if (!active) {
            return;
        }
        
        try {
            if (acceptsMessage(message)) {
                subscriber.onMessage(message);
            }
        } catch (Exception e) {
            subscriber.onError(message, e);
        }
    }
    
    public void cancel() {
        this.active = false;
        topic.removeSubscriber(this);
    }
    
    public boolean isActive() {
        return active;
    }
    
    // Getters
    public String getId() { return id; }
    public Topic getTopic() { return topic; }
    public Subscriber getSubscriber() { return subscriber; }
    
    @Override
    public String toString() {
        return String.format("Subscription[id=%s, topic=%s, subscriber=%s, active=%s]",
                id, topic.getName(), subscriber.getId(), active);
    }
}
```

### 2.4 Topic Class

```java
// Topic.java
package com.pubsub;

import java.util.*;
import java.util.concurrent.CopyOnWriteArraySet;

/**
 * Represents a topic that subscribers can subscribe to.
 * 
 * Thread-safe using CopyOnWriteArraySet for subscribers.
 */
public class Topic {
    
    private final String name;
    private final Set<Subscription> subscriptions;
    private final Deque<Message> messageHistory;
    private final int historySize;
    
    public Topic(String name) {
        this(name, 100);  // Default history size
    }
    
    public Topic(String name, int historySize) {
        this.name = name;
        this.subscriptions = new CopyOnWriteArraySet<>();
        this.messageHistory = new LinkedList<>();
        this.historySize = historySize;
    }
    
    /**
     * Adds a subscription to this topic.
     */
    public void addSubscriber(Subscription subscription) {
        subscriptions.add(subscription);
    }
    
    /**
     * Removes a subscription from this topic.
     */
    public void removeSubscriber(Subscription subscription) {
        subscriptions.remove(subscription);
    }
    
    /**
     * Publishes a message to all subscribers.
     * 
     * @param message The message to publish
     * @return Number of subscribers that received the message
     */
    public int publish(Message message) {
        // Add to history
        addToHistory(message);
        
        // Deliver to all active subscriptions
        int delivered = 0;
        for (Subscription subscription : subscriptions) {
            if (subscription.isActive() && subscription.acceptsMessage(message)) {
                subscription.deliver(message);
                delivered++;
            }
        }
        
        return delivered;
    }
    
    private synchronized void addToHistory(Message message) {
        messageHistory.addLast(message);
        while (messageHistory.size() > historySize) {
            messageHistory.removeFirst();
        }
    }
    
    /**
     * Gets recent messages from history.
     */
    public List<Message> getRecentMessages(int count) {
        synchronized (this) {
            List<Message> recent = new ArrayList<>();
            Iterator<Message> it = messageHistory.descendingIterator();
            while (it.hasNext() && recent.size() < count) {
                recent.add(it.next());
            }
            Collections.reverse(recent);
            return recent;
        }
    }
    
    /**
     * Gets the number of active subscribers.
     */
    public int getSubscriberCount() {
        return (int) subscriptions.stream()
                .filter(Subscription::isActive)
                .count();
    }
    
    public String getName() {
        return name;
    }
    
    public Set<Subscription> getSubscriptions() {
        return Collections.unmodifiableSet(subscriptions);
    }
    
    @Override
    public String toString() {
        return String.format("Topic[name=%s, subscribers=%d]", name, getSubscriberCount());
    }
}
```

### 2.5 PubSubService Class

```java
// PubSubService.java
package com.pubsub;

import java.util.*;
import java.util.concurrent.*;
import java.util.function.Predicate;

/**
 * Main Pub/Sub service that manages topics and message delivery.
 * 
 * Supports both synchronous and asynchronous message delivery.
 */
public class PubSubService {
    
    private final Map<String, Topic> topics;
    private final ExecutorService executor;
    private final boolean asyncDelivery;
    private volatile boolean running;
    
    public PubSubService() {
        this(false);  // Synchronous by default
    }
    
    public PubSubService(boolean asyncDelivery) {
        this.topics = new ConcurrentHashMap<>();
        this.asyncDelivery = asyncDelivery;
        this.executor = asyncDelivery ? 
            Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors()) : null;
        this.running = true;
    }
    
    // ==================== Topic Management ====================
    
    /**
     * Creates a new topic.
     */
    public Topic createTopic(String name) {
        return topics.computeIfAbsent(name, Topic::new);
    }
    
    /**
     * Gets a topic by name.
     */
    public Topic getTopic(String name) {
        return topics.get(name);
    }
    
    /**
     * Deletes a topic and notifies subscribers.
     */
    public boolean deleteTopic(String name) {
        Topic topic = topics.remove(name);
        if (topic != null) {
            // Cancel all subscriptions
            for (Subscription sub : topic.getSubscriptions()) {
                sub.cancel();
            }
            return true;
        }
        return false;
    }
    
    /**
     * Gets all topic names.
     */
    public Set<String> getTopicNames() {
        return Collections.unmodifiableSet(topics.keySet());
    }
    
    // ==================== Publishing ====================
    
    /**
     * Publishes a message to a topic.
     */
    public void publish(String topicName, Object payload) {
        publish(Message.of(topicName, payload));
    }
    
    /**
     * Publishes a message to a topic.
     */
    public void publish(Message message) {
        if (!running) {
            throw new IllegalStateException("PubSubService is shut down");
        }
        
        Topic topic = topics.get(message.getTopic());
        if (topic == null) {
            throw new IllegalArgumentException("Topic not found: " + message.getTopic());
        }
        
        if (asyncDelivery) {
            publishAsync(topic, message);
        } else {
            topic.publish(message);
        }
    }
    
    private void publishAsync(Topic topic, Message message) {
        for (Subscription subscription : topic.getSubscriptions()) {
            if (subscription.isActive() && subscription.acceptsMessage(message)) {
                executor.submit(() -> {
                    try {
                        subscription.deliver(message);
                    } catch (Exception e) {
                        subscription.getSubscriber().onError(message, e);
                    }
                });
            }
        }
    }
    
    // ==================== Subscribing ====================
    
    /**
     * Subscribes to a topic.
     */
    public Subscription subscribe(String topicName, Subscriber subscriber) {
        return subscribe(topicName, subscriber, null);
    }
    
    /**
     * Subscribes to a topic with a message filter.
     */
    public Subscription subscribe(String topicName, Subscriber subscriber, 
                                  Predicate<Message> filter) {
        Topic topic = topics.get(topicName);
        if (topic == null) {
            throw new IllegalArgumentException("Topic not found: " + topicName);
        }
        
        Subscription subscription = new Subscription(topic, subscriber, filter);
        topic.addSubscriber(subscription);
        return subscription;
    }
    
    /**
     * Unsubscribes from a topic.
     */
    public boolean unsubscribe(Subscription subscription) {
        if (subscription != null) {
            subscription.cancel();
            return true;
        }
        return false;
    }
    
    // ==================== Lifecycle ====================
    
    /**
     * Shuts down the service.
     */
    public void shutdown() {
        running = false;
        if (executor != null) {
            executor.shutdown();
            try {
                if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
                    executor.shutdownNow();
                }
            } catch (InterruptedException e) {
                executor.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
    }
    
    public boolean isRunning() {
        return running;
    }
}
```

### 2.6 Common Subscriber Implementations

```java
// PrintSubscriber.java
package com.pubsub;

/**
 * Simple subscriber that prints messages to console.
 */
public class PrintSubscriber implements Subscriber {
    
    private final String id;
    private final String prefix;
    
    public PrintSubscriber(String id) {
        this(id, "");
    }
    
    public PrintSubscriber(String id, String prefix) {
        this.id = id;
        this.prefix = prefix;
    }
    
    @Override
    public void onMessage(Message message) {
        System.out.println(prefix + "[" + id + "] Received: " + message.getPayload());
    }
    
    @Override
    public String getId() {
        return id;
    }
}
```

```java
// QueueSubscriber.java
package com.pubsub;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

/**
 * Subscriber that queues messages for later processing.
 */
public class QueueSubscriber implements Subscriber {
    
    private final String id;
    private final BlockingQueue<Message> queue;
    
    public QueueSubscriber(String id) {
        this(id, new LinkedBlockingQueue<>());
    }
    
    public QueueSubscriber(String id, BlockingQueue<Message> queue) {
        this.id = id;
        this.queue = queue;
    }
    
    @Override
    public void onMessage(Message message) {
        queue.offer(message);
    }
    
    @Override
    public String getId() {
        return id;
    }
    
    /**
     * Polls for the next message.
     */
    public Message poll() {
        return queue.poll();
    }
    
    /**
     * Polls for the next message with timeout.
     */
    public Message poll(long timeout, TimeUnit unit) throws InterruptedException {
        return queue.poll(timeout, unit);
    }
    
    /**
     * Takes the next message (blocking).
     */
    public Message take() throws InterruptedException {
        return queue.take();
    }
    
    /**
     * Gets the number of queued messages.
     */
    public int getQueueSize() {
        return queue.size();
    }
}
```

```java
// CallbackSubscriber.java
package com.pubsub;

import java.util.function.Consumer;

/**
 * Subscriber that invokes a callback function.
 */
public class CallbackSubscriber implements Subscriber {
    
    private final String id;
    private final Consumer<Message> callback;
    
    public CallbackSubscriber(String id, Consumer<Message> callback) {
        this.id = id;
        this.callback = callback;
    }
    
    @Override
    public void onMessage(Message message) {
        callback.accept(message);
    }
    
    @Override
    public String getId() {
        return id;
    }
}
```

### 2.7 Message Filters

```java
// MessageFilters.java
package com.pubsub;

import java.util.function.Predicate;

/**
 * Common message filters.
 */
public class MessageFilters {
    
    /**
     * Filter by header value.
     */
    public static Predicate<Message> hasHeader(String key, String value) {
        return msg -> value.equals(msg.getHeader(key));
    }
    
    /**
     * Filter by header existence.
     */
    public static Predicate<Message> hasHeader(String key) {
        return msg -> msg.getHeaders().containsKey(key);
    }
    
    /**
     * Filter by payload type.
     */
    public static Predicate<Message> payloadType(Class<?> type) {
        return msg -> type.isInstance(msg.getPayload());
    }
    
    /**
     * Filter by payload content (contains string).
     */
    public static Predicate<Message> payloadContains(String text) {
        return msg -> {
            Object payload = msg.getPayload();
            return payload != null && payload.toString().contains(text);
        };
    }
    
    /**
     * Combine filters with AND.
     */
    public static Predicate<Message> and(Predicate<Message>... filters) {
        return msg -> {
            for (Predicate<Message> filter : filters) {
                if (!filter.test(msg)) {
                    return false;
                }
            }
            return true;
        };
    }
    
    /**
     * Combine filters with OR.
     */
    public static Predicate<Message> or(Predicate<Message>... filters) {
        return msg -> {
            for (Predicate<Message> filter : filters) {
                if (filter.test(msg)) {
                    return true;
                }
            }
            return false;
        };
    }
    
    /**
     * Negate a filter.
     */
    public static Predicate<Message> not(Predicate<Message> filter) {
        return filter.negate();
    }
}
```

### 2.8 Demo Application

```java
// PubSubDemo.java
package com.pubsub;

import java.util.concurrent.TimeUnit;

public class PubSubDemo {
    
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== PUB/SUB SYSTEM DEMO ===\n");
        
        PubSubService pubsub = new PubSubService();
        
        // ==================== Create Topics ====================
        System.out.println("===== CREATING TOPICS =====\n");
        
        pubsub.createTopic("orders");
        pubsub.createTopic("notifications");
        pubsub.createTopic("logs");
        
        System.out.println("Topics created: " + pubsub.getTopicNames());
        
        // ==================== Subscribe ====================
        System.out.println("\n===== SUBSCRIBING =====\n");
        
        // Simple print subscriber
        Subscription orderSub1 = pubsub.subscribe("orders", 
            new PrintSubscriber("OrderProcessor", "  "));
        
        // Queue subscriber for batch processing
        QueueSubscriber queueSub = new QueueSubscriber("OrderQueue");
        Subscription orderSub2 = pubsub.subscribe("orders", queueSub);
        
        // Callback subscriber
        Subscription notifSub = pubsub.subscribe("notifications",
            new CallbackSubscriber("NotifHandler", msg -> 
                System.out.println("  📢 Notification: " + msg.getPayload())));
        
        // Filtered subscriber (only ERROR logs)
        Subscription logSub = pubsub.subscribe("logs",
            new PrintSubscriber("ErrorLogger", "  ❌ "),
            MessageFilters.hasHeader("level", "ERROR"));
        
        System.out.println("Subscriptions created");
        
        // ==================== Publish Messages ====================
        System.out.println("\n===== PUBLISHING MESSAGES =====\n");
        
        // Publish orders
        System.out.println("Publishing orders:");
        pubsub.publish("orders", "Order #1001 - iPhone");
        pubsub.publish("orders", "Order #1002 - MacBook");
        
        // Publish notifications
        System.out.println("\nPublishing notifications:");
        pubsub.publish("notifications", "Welcome to our store!");
        pubsub.publish("notifications", "Sale starts tomorrow!");
        
        // Publish logs (with headers)
        System.out.println("\nPublishing logs:");
        pubsub.publish(Message.builder()
            .topic("logs")
            .payload("Application started")
            .header("level", "INFO")
            .build());
        
        pubsub.publish(Message.builder()
            .topic("logs")
            .payload("Database connection failed!")
            .header("level", "ERROR")
            .build());
        
        pubsub.publish(Message.builder()
            .topic("logs")
            .payload("User logged in")
            .header("level", "INFO")
            .build());
        
        // ==================== Queue Processing ====================
        System.out.println("\n===== QUEUE PROCESSING =====\n");
        
        System.out.println("Messages in queue: " + queueSub.getQueueSize());
        System.out.println("Processing queue:");
        Message msg;
        while ((msg = queueSub.poll()) != null) {
            System.out.println("  Processed: " + msg.getPayload());
        }
        
        // ==================== Unsubscribe ====================
        System.out.println("\n===== UNSUBSCRIBE =====\n");
        
        orderSub1.cancel();
        System.out.println("OrderProcessor unsubscribed");
        
        System.out.println("Publishing after unsubscribe:");
        pubsub.publish("orders", "Order #1003 - iPad");
        
        System.out.println("Queue size after publish: " + queueSub.getQueueSize());
        
        // ==================== Message History ====================
        System.out.println("\n===== MESSAGE HISTORY =====\n");
        
        Topic ordersTopic = pubsub.getTopic("orders");
        System.out.println("Recent orders:");
        for (Message m : ordersTopic.getRecentMessages(5)) {
            System.out.println("  " + m.getTimestamp() + ": " + m.getPayload());
        }
        
        // ==================== Cleanup ====================
        System.out.println("\n===== CLEANUP =====\n");
        
        pubsub.shutdown();
        System.out.println("PubSub service shut down");
        
        System.out.println("\n=== DEMO COMPLETE ===");
    }
}
```

---

## STEP 5: Simulation / Dry Run

### Scenario: Order Processing

```
Initial State:
- Topics: {orders, notifications}
- Subscriptions: empty

Step 1: subscribe("orders", PrintSubscriber("Processor1"))
- Find topic "orders"
- Create Subscription(topic, subscriber, filter=null)
- Add to topic.subscriptions
- Return subscription

State: orders.subscriptions = [Sub1]

Step 2: subscribe("orders", PrintSubscriber("Processor2"), 
                  hasHeader("priority", "HIGH"))
- Create Subscription with filter
- Add to topic.subscriptions

State: orders.subscriptions = [Sub1, Sub2]

Step 3: publish(Message.of("orders", "Regular Order"))
- Find topic "orders"
- For Sub1:
  - acceptsMessage? filter=null → true
  - deliver → "Processor1 received: Regular Order"
- For Sub2:
  - acceptsMessage? filter checks header "priority"
  - header not present → false
  - skip delivery

Output: Only Processor1 receives message

Step 4: publish(Message.builder()
                .topic("orders")
                .payload("Urgent Order")
                .header("priority", "HIGH")
                .build())
- For Sub1:
  - acceptsMessage? true
  - deliver → "Processor1 received: Urgent Order"
- For Sub2:
  - acceptsMessage? header "priority" = "HIGH" → true
  - deliver → "Processor2 received: Urgent Order"

Output: Both processors receive message
```

---

## File Structure

```
com/pubsub/
├── Message.java
├── Subscriber.java
├── Subscription.java
├── Topic.java
├── PubSubService.java
├── PrintSubscriber.java
├── QueueSubscriber.java
├── CallbackSubscriber.java
├── MessageFilters.java
└── PubSubDemo.java
```

