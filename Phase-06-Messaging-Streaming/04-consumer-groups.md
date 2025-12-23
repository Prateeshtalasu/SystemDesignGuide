# 👥 Consumer Groups

---

## 0️⃣ Prerequisites

Before diving into consumer groups, you should understand:

- **Queue vs Pub/Sub** (Topic 1): Queue delivers to one consumer, Pub/Sub delivers to all subscribers.
- **Message Delivery** (Topic 2): At-least-once delivery and acknowledgment patterns.
- **Partitions**: A way to split a topic into multiple ordered sequences. Each partition is an independent, ordered log of messages. Think of it like lanes on a highway, each lane handles traffic independently.
- **Offset**: A sequential number assigned to each message within a partition. It's like a bookmark that tracks your position in the message log.

**Quick refresher on scaling consumers**: If you have one consumer processing messages, and messages arrive faster than the consumer can process, you have a backlog problem. The obvious solution is to add more consumers. But how do you coordinate multiple consumers so they don't process the same message twice? That's where consumer groups come in.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

Imagine you have a topic receiving 10,000 messages per second, but each consumer can only process 1,000 messages per second.

**Without consumer groups:**

```
┌─────────────────────────────────────────────────────────────┐
│                    THE SCALING PROBLEM                       │
│                                                              │
│   Topic: 10,000 msg/sec                                     │
│          │                                                   │
│          ▼                                                   │
│   ┌──────────────┐                                          │
│   │  Consumer 1  │  Can only process 1,000 msg/sec          │
│   └──────────────┘                                          │
│                                                              │
│   Result: 9,000 messages/sec backlog!                       │
│   Queue grows by 9,000 messages every second.               │
│   After 1 hour: 32.4 million message backlog!               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Naive approach: Add more consumers**

```
┌─────────────────────────────────────────────────────────────┐
│                    NAIVE SCALING                             │
│                                                              │
│   Topic: 10,000 msg/sec                                     │
│          │                                                   │
│          ├──────────► Consumer 1 (gets ALL messages)        │
│          ├──────────► Consumer 2 (gets ALL messages)        │
│          └──────────► Consumer 3 (gets ALL messages)        │
│                                                              │
│   Problem: Each message processed 3 times!                  │
│   This is Pub/Sub behavior, not what we want.               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**What we actually want:**

```
┌─────────────────────────────────────────────────────────────┐
│                    DESIRED BEHAVIOR                          │
│                                                              │
│   Topic: 10,000 msg/sec                                     │
│          │                                                   │
│          ├──────────► Consumer 1 (gets ~3,333 msg/sec)      │
│          ├──────────► Consumer 2 (gets ~3,333 msg/sec)      │
│          └──────────► Consumer 3 (gets ~3,333 msg/sec)      │
│                                                              │
│   Each message processed exactly once (by one consumer).    │
│   Work is distributed evenly.                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### What Systems Looked Like Before Consumer Groups

Before consumer groups, scaling consumers required:

1. **Manual partitioning**: Application code decided which consumer handles which messages.
2. **External coordination**: Using ZooKeeper or database locks to assign work.
3. **Static assignment**: Consumer 1 always handles A-M, Consumer 2 handles N-Z.
4. **Custom load balancing**: Building your own work distribution logic.

All of these were error-prone, hard to scale, and didn't handle consumer failures well.

### What Breaks Without Consumer Groups

1. **No automatic load balancing**: Adding a consumer doesn't automatically distribute work.

2. **No failure handling**: If a consumer dies, its messages aren't reassigned.

3. **Duplicate processing**: Multiple consumers might process the same message.

4. **Manual offset management**: Each consumer must track its own position.

5. **Scaling complexity**: Adding/removing consumers requires code changes.

### Real Examples of the Problem

**LinkedIn (Kafka's Origin)**:
LinkedIn created Kafka and consumer groups because they needed to process billions of events per day. A single consumer couldn't keep up, and they needed automatic scaling and failure recovery.

**Uber's Trip Events**:
Uber processes millions of trip events. Multiple services need to consume these events, and each service needs multiple consumers for throughput. Consumer groups let them scale each service independently.

**Netflix's Viewing History**:
Netflix records every play, pause, and stop. They need dozens of consumers to keep up with the volume, and consumer groups ensure each event is processed exactly once per service.

---

## 2️⃣ Intuition and Mental Model

### The Restaurant Kitchen Analogy

Think of a consumer group like a kitchen staff handling orders:

**Without Consumer Groups: One Chef**
```
┌─────────────────────────────────────────────────────────────┐
│                    ONE CHEF KITCHEN                          │
│                                                              │
│   Orders: [Burger, Pizza, Salad, Pasta, Steak, ...]        │
│              │                                               │
│              ▼                                               │
│        ┌──────────┐                                         │
│        │  Chef 1  │  Makes ALL dishes                       │
│        └──────────┘  (overwhelmed!)                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**With Consumer Groups: Kitchen Staff**
```
┌─────────────────────────────────────────────────────────────┐
│                  KITCHEN WITH STAFF                          │
│                                                              │
│   Orders come in on 4 order tickets (partitions):           │
│   [Ticket 1] [Ticket 2] [Ticket 3] [Ticket 4]               │
│                                                              │
│   Kitchen Staff (Consumer Group: "kitchen"):                │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│   │  Chef 1  │  │  Chef 2  │  │  Chef 3  │  │  Chef 4  │   │
│   │ Ticket 1 │  │ Ticket 2 │  │ Ticket 3 │  │ Ticket 4 │   │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│                                                              │
│   Each chef handles their own tickets.                      │
│   No chef works on another chef's tickets.                  │
│   If Chef 2 gets sick, Chef 1 takes over Ticket 2.         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Multiple Consumer Groups: Kitchen + Delivery**
```
┌─────────────────────────────────────────────────────────────┐
│              KITCHEN + DELIVERY TEAMS                        │
│                                                              │
│   Same orders go to BOTH teams:                             │
│                                                              │
│   Consumer Group: "kitchen"                                 │
│   ┌──────────┐  ┌──────────┐                                │
│   │  Chef 1  │  │  Chef 2  │   (prepare food)               │
│   └──────────┘  └──────────┘                                │
│                                                              │
│   Consumer Group: "delivery"                                │
│   ┌──────────┐  ┌──────────┐                                │
│   │ Driver 1 │  │ Driver 2 │   (plan routes)                │
│   └──────────┘  └──────────┘                                │
│                                                              │
│   Kitchen team processes orders (cooking).                  │
│   Delivery team processes orders (routing).                 │
│   Each team gets ALL orders, but distributes internally.    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### The Core Concept

```
┌─────────────────────────────────────────────────────────────┐
│              CONSUMER GROUP MENTAL MODEL                     │
│                                                              │
│   TOPIC (with partitions)                                   │
│   ┌─────────────────────────────────────────────────┐       │
│   │ Partition 0: [M1] [M4] [M7] [M10] ...           │       │
│   │ Partition 1: [M2] [M5] [M8] [M11] ...           │       │
│   │ Partition 2: [M3] [M6] [M9] [M12] ...           │       │
│   └─────────────────────────────────────────────────┘       │
│                                                              │
│   CONSUMER GROUP "order-processor"                          │
│   ┌─────────────────────────────────────────────────┐       │
│   │ Consumer A ←── Partition 0                      │       │
│   │ Consumer B ←── Partition 1                      │       │
│   │ Consumer C ←── Partition 2                      │       │
│   └─────────────────────────────────────────────────┘       │
│                                                              │
│   KEY RULES:                                                 │
│   • Each partition → exactly one consumer in the group     │
│   • Each consumer → can handle multiple partitions         │
│   • Messages in partition → processed in order             │
│   • Different groups → each gets all messages              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ How It Works Internally

### Partition Assignment

When consumers join a group, the broker assigns partitions to consumers.

**Scenario: 6 partitions, 3 consumers**

```
Initial State:
┌─────────────────────────────────────────────────────────────┐
│   Topic: orders (6 partitions)                              │
│   Consumer Group: order-processor                           │
│                                                              │
│   Partition 0 ─────► Consumer A                             │
│   Partition 1 ─────► Consumer A                             │
│   Partition 2 ─────► Consumer B                             │
│   Partition 3 ─────► Consumer B                             │
│   Partition 4 ─────► Consumer C                             │
│   Partition 5 ─────► Consumer C                             │
│                                                              │
│   Each consumer handles 2 partitions.                       │
│   Work is evenly distributed.                               │
└─────────────────────────────────────────────────────────────┘
```

**Scenario: Consumer B dies (Rebalancing)**

```
After Consumer B fails:
┌─────────────────────────────────────────────────────────────┐
│   Partition 0 ─────► Consumer A                             │
│   Partition 1 ─────► Consumer A                             │
│   Partition 2 ─────► Consumer A  (reassigned from B)        │
│   Partition 3 ─────► Consumer C  (reassigned from B)        │
│   Partition 4 ─────► Consumer C                             │
│   Partition 5 ─────► Consumer C                             │
│                                                              │
│   Consumer A: 3 partitions                                  │
│   Consumer C: 3 partitions                                  │
│   No messages lost!                                         │
└─────────────────────────────────────────────────────────────┘
```

### The Rebalancing Process

Rebalancing happens when:
- Consumer joins the group
- Consumer leaves the group (graceful shutdown)
- Consumer fails (heartbeat timeout)
- Partitions added to topic

**Rebalancing Steps:**

```
┌─────────────────────────────────────────────────────────────┐
│                    REBALANCING PROCESS                       │
│                                                              │
│   1. TRIGGER: Consumer C joins the group                    │
│                                                              │
│   2. STOP: All consumers stop processing                    │
│      (brief pause in message consumption)                   │
│                                                              │
│   3. REVOKE: Consumers give up their partitions             │
│      Consumer A: "I release P0, P1, P2"                     │
│      Consumer B: "I release P3, P4, P5"                     │
│                                                              │
│   4. ASSIGN: Coordinator reassigns partitions               │
│      Consumer A: assigned P0, P1                            │
│      Consumer B: assigned P2, P3                            │
│      Consumer C: assigned P4, P5                            │
│                                                              │
│   5. RESUME: Consumers start processing new assignments     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Offset Management

Each consumer group tracks its position (offset) in each partition independently.

**Offset Storage:**

```
┌─────────────────────────────────────────────────────────────┐
│              OFFSET TRACKING                                 │
│                                                              │
│   __consumer_offsets topic (internal Kafka topic):          │
│                                                              │
│   Key: (group_id, topic, partition)                         │
│   Value: offset                                              │
│                                                              │
│   Examples:                                                  │
│   (order-processor, orders, 0) → 1547                       │
│   (order-processor, orders, 1) → 2891                       │
│   (order-processor, orders, 2) → 1203                       │
│                                                              │
│   (analytics-consumer, orders, 0) → 892                     │
│   (analytics-consumer, orders, 1) → 1456                    │
│                                                              │
│   Different groups have different offsets!                  │
│   Each group progresses independently.                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Commit Strategies

**Auto Commit:**
```
Consumer polls messages
  → processes
  → auto-commit offset periodically (default: 5 seconds)

Risk: If consumer crashes after processing but before commit,
      messages will be reprocessed (at-least-once).
```

**Manual Commit (Synchronous):**
```java
while (true) {
    records = consumer.poll(Duration.ofMillis(100));
    for (record : records) {
        process(record);
    }
    consumer.commitSync();  // Block until committed
}

Pros: More control, know exactly when committed
Cons: Slower (waits for commit acknowledgment)
```

**Manual Commit (Asynchronous):**
```java
while (true) {
    records = consumer.poll(Duration.ofMillis(100));
    for (record : records) {
        process(record);
    }
    consumer.commitAsync();  // Don't wait
}

Pros: Faster (doesn't block)
Cons: If commit fails, you might not know
```

---

## 4️⃣ Simulation-First Explanation

Let's trace through consumer group behavior step by step.

### Scenario: Order Processing System

**Setup:**
- Topic: `orders` with 4 partitions
- Consumer Group: `order-processor`
- Initial: 2 consumers (A and B)

### Initial State

```
Topic: orders
┌─────────────────────────────────────────────────────────────┐
│ Partition 0: [O1] [O5] [O9]  [O13]                         │
│ Partition 1: [O2] [O6] [O10] [O14]                         │
│ Partition 2: [O3] [O7] [O11] [O15]                         │
│ Partition 3: [O4] [O8] [O12] [O16]                         │
└─────────────────────────────────────────────────────────────┘

Consumer Group: order-processor
┌─────────────────────────────────────────────────────────────┐
│ Consumer A: Partitions 0, 1                                 │
│   Current offsets: P0=0, P1=0                              │
│                                                              │
│ Consumer B: Partitions 2, 3                                 │
│   Current offsets: P2=0, P3=0                              │
└─────────────────────────────────────────────────────────────┘
```

### Processing Flow

**Time 0ms: Consumer A polls**
```
Consumer A polls from P0 and P1:
  - Gets: O1 (P0), O2 (P1)
  - Processes O1, O2
  - Commits: P0=1, P1=1

Current offsets:
  Consumer A: P0=1, P1=1
  Consumer B: P2=0, P3=0
```

**Time 100ms: Consumer B polls**
```
Consumer B polls from P2 and P3:
  - Gets: O3 (P2), O4 (P3)
  - Processes O3, O4
  - Commits: P2=1, P3=1

Current offsets:
  Consumer A: P0=1, P1=1
  Consumer B: P2=1, P3=1
```

**Time 200ms: Consumer C joins (Rebalancing)**
```
1. Coordinator detects new consumer C
2. Triggers rebalance
3. All consumers pause

New assignment:
  Consumer A: Partition 0, 1
  Consumer B: Partition 2
  Consumer C: Partition 3

4. Consumer C reads committed offset for P3: offset=1
5. Consumer C starts from O8 (offset 1 in P3)
```

**Time 300ms: Consumer B crashes**
```
1. Consumer B stops sending heartbeats
2. After session.timeout.ms (default 10s), coordinator detects failure
3. Triggers rebalance

New assignment:
  Consumer A: Partition 0, 1
  Consumer C: Partition 2, 3

4. Consumer C takes over P2
5. Consumer C reads committed offset for P2: offset=1
6. Consumer C continues from where B left off
```

### Multiple Consumer Groups

```
Topic: orders (same messages)

Consumer Group: order-processor
┌─────────────────────────────────────────────────────────────┐
│ Consumer A: P0, P1 → Processing orders                     │
│ Consumer B: P2, P3 → Processing orders                     │
│ Offsets: P0=100, P1=150, P2=120, P3=130                   │
└─────────────────────────────────────────────────────────────┘

Consumer Group: analytics
┌─────────────────────────────────────────────────────────────┐
│ Consumer X: P0, P1, P2, P3 → Tracking metrics              │
│ Offsets: P0=50, P1=75, P2=60, P3=80                       │
│ (behind order-processor, processing slower)                │
└─────────────────────────────────────────────────────────────┘

Consumer Group: fraud-detection
┌─────────────────────────────────────────────────────────────┐
│ Consumer Y: P0, P1 → Checking for fraud                    │
│ Consumer Z: P2, P3 → Checking for fraud                    │
│ Offsets: P0=100, P1=150, P2=120, P3=130                   │
│ (same as order-processor, keeping up)                      │
└─────────────────────────────────────────────────────────────┘

Each group processes ALL messages independently.
Each group has its own offset tracking.
```

---

## 5️⃣ How Engineers Actually Use This in Production

### LinkedIn's Usage

LinkedIn (Kafka's creator) uses consumer groups extensively:

- **Activity tracking**: Multiple consumer groups process user activity (views, clicks, etc.)
- **Search indexing**: Consumer group updates search index from activity stream
- **Recommendations**: Consumer group feeds recommendation engine
- **Analytics**: Consumer group aggregates metrics

Each service has its own consumer group, processing the same events for different purposes.

### Uber's Architecture

Uber uses consumer groups for:

1. **Trip events**: Multiple services consume trip lifecycle events
   - Billing service (consumer group: `billing`)
   - Analytics service (consumer group: `analytics`)
   - Driver payments (consumer group: `driver-payments`)

2. **Scaling**: Each service scales its consumer group independently
   - Billing: 10 consumers (high volume)
   - Analytics: 5 consumers (can be delayed)
   - Driver payments: 20 consumers (critical, needs low latency)

### Netflix's Approach

Netflix uses consumer groups for:

1. **Viewing history**: Consumer group processes play/pause/stop events
2. **Recommendations**: Separate consumer group feeds ML models
3. **A/B testing**: Consumer group tracks experiment results

Key practices:
- Each microservice has dedicated consumer group
- Consumer count matches partition count for optimal parallelism
- Separate consumer groups for real-time vs batch processing

### Airbnb's Event Processing

Airbnb's pattern:

```
Event: Booking Created

Consumer Groups:
├── booking-confirmation (sends email)
├── host-notification (notifies host)
├── calendar-sync (updates availability)
├── analytics (tracks metrics)
├── fraud-detection (checks for fraud)
└── search-index (updates search)

Each group processes the same event for different purposes.
```

---

## 6️⃣ How to Implement or Apply It

### Maven Dependencies

```xml
<dependencies>
    <!-- Spring Kafka -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

### Basic Consumer Group Configuration

```java
package com.systemdesign.messaging.consumergroups;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.support.serializer.JsonDeserializer;

import java.util.HashMap;
import java.util.Map;

/**
 * Kafka consumer configuration with consumer groups.
 */
@Configuration
@EnableKafka
public class KafkaConsumerConfig {
    
    /**
     * Consumer factory with group configuration.
     */
    @Bean
    public ConsumerFactory<String, Order> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        
        // Bootstrap servers
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        
        // Consumer group ID - THIS IS THE KEY SETTING
        // All consumers with same group.id share the partitions
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processor");
        
        // Deserializers
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
            StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
            JsonDeserializer.class);
        
        // Auto offset reset: what to do when no committed offset exists
        // "earliest" = start from beginning
        // "latest" = start from end (only new messages)
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        
        // Auto commit settings
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 5000);
        
        // Session timeout: how long before consumer considered dead
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 10000);
        
        // Heartbeat interval: how often to send heartbeat
        props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 3000);
        
        // Max poll interval: max time between polls before considered dead
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000);
        
        // Max records per poll
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
        
        return new DefaultKafkaConsumerFactory<>(props, 
            new StringDeserializer(),
            new JsonDeserializer<>(Order.class));
    }
    
    /**
     * Listener container factory.
     * Concurrency setting creates multiple consumers in the same group.
     */
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Order> 
            kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Order> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        
        // CONCURRENCY: Number of consumer threads
        // Each thread is a separate consumer in the group
        // Set to number of partitions for optimal parallelism
        factory.setConcurrency(4);
        
        return factory;
    }
}
```

### Consumer with Manual Offset Management

```java
package com.systemdesign.messaging.consumergroups;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Service;

/**
 * Order consumer with manual offset management.
 * Part of consumer group "order-processor".
 */
@Service
public class OrderConsumer {
    
    private final OrderService orderService;
    
    public OrderConsumer(OrderService orderService) {
        this.orderService = orderService;
    }
    
    /**
     * Processes orders from the orders topic.
     * Multiple instances of this consumer share the partitions.
     * 
     * @param record The order record
     * @param ack Acknowledgment for manual commit
     */
    @KafkaListener(
        topics = "orders",
        groupId = "order-processor",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void processOrder(ConsumerRecord<String, Order> record,
                            Acknowledgment ack) {
        
        Order order = record.value();
        
        System.out.println(String.format(
            "Consumer received order: %s from partition: %d, offset: %d",
            order.getId(),
            record.partition(),
            record.offset()
        ));
        
        try {
            // Process the order
            orderService.processOrder(order);
            
            // Acknowledge (commit offset)
            ack.acknowledge();
            
            System.out.println("Order processed and committed: " + order.getId());
            
        } catch (Exception e) {
            System.err.println("Failed to process order: " + e.getMessage());
            // Don't acknowledge - message will be redelivered
            // Or send to DLQ
            throw e;
        }
    }
}
```

### Multiple Consumer Groups for Same Topic

```java
package com.systemdesign.messaging.consumergroups;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

/**
 * Multiple consumer groups processing the same topic.
 * Each group gets ALL messages independently.
 */

// Consumer Group 1: Order Processing
@Service
public class OrderProcessingConsumer {
    
    @KafkaListener(
        topics = "orders",
        groupId = "order-processor"  // Group 1
    )
    public void processOrder(ConsumerRecord<String, Order> record) {
        System.out.println("[ORDER-PROCESSOR] Processing: " + record.value().getId());
        // Fulfill the order
    }
}

// Consumer Group 2: Analytics
@Service
public class AnalyticsConsumer {
    
    @KafkaListener(
        topics = "orders",
        groupId = "analytics"  // Group 2 - different group!
    )
    public void trackOrder(ConsumerRecord<String, Order> record) {
        System.out.println("[ANALYTICS] Tracking: " + record.value().getId());
        // Update metrics, dashboards
    }
}

// Consumer Group 3: Fraud Detection
@Service
public class FraudDetectionConsumer {
    
    @KafkaListener(
        topics = "orders",
        groupId = "fraud-detection"  // Group 3 - different group!
    )
    public void checkFraud(ConsumerRecord<String, Order> record) {
        System.out.println("[FRAUD] Checking: " + record.value().getId());
        // Run fraud detection
    }
}

/*
 * Result:
 * - Each order is processed by order-processor (one consumer in group)
 * - Each order is tracked by analytics (one consumer in group)
 * - Each order is checked by fraud-detection (one consumer in group)
 * 
 * Three different purposes, same data source.
 */
```

### Handling Rebalances

```java
package com.systemdesign.messaging.consumergroups;

import org.apache.kafka.clients.consumer.ConsumerRebalanceListener;
import org.apache.kafka.common.TopicPartition;
import org.springframework.kafka.listener.ConsumerAwareRebalanceListener;
import org.springframework.stereotype.Component;

import java.util.Collection;

/**
 * Listener for consumer group rebalance events.
 * Useful for cleanup and state management during rebalances.
 */
@Component
public class RebalanceListener implements ConsumerAwareRebalanceListener {
    
    /**
     * Called when partitions are revoked (before rebalance).
     * Use this to commit offsets and clean up state.
     */
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        System.out.println("Partitions revoked: " + partitions);
        
        for (TopicPartition partition : partitions) {
            // Clean up any state for this partition
            // For example: flush in-memory buffers, commit pending work
            System.out.println("Cleaning up partition: " + partition);
        }
    }
    
    /**
     * Called when partitions are assigned (after rebalance).
     * Use this to initialize state for new partitions.
     */
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        System.out.println("Partitions assigned: " + partitions);
        
        for (TopicPartition partition : partitions) {
            // Initialize state for this partition
            // For example: load checkpoint, initialize counters
            System.out.println("Initializing partition: " + partition);
        }
    }
    
    /**
     * Called when partitions are lost (consumer failed to heartbeat).
     * Different from revoked - no time to clean up.
     */
    @Override
    public void onPartitionsLost(Collection<TopicPartition> partitions) {
        System.out.println("Partitions lost (unexpected): " + partitions);
        // State may be inconsistent - handle carefully
    }
}
```

### Configuration for Rebalancing Strategies

```yaml
# application.yml
spring:
  kafka:
    consumer:
      group-id: order-processor
      auto-offset-reset: earliest
      enable-auto-commit: false  # Manual commit for more control
      
      properties:
        # Session management
        session.timeout.ms: 10000
        heartbeat.interval.ms: 3000
        max.poll.interval.ms: 300000
        
        # Partition assignment strategy
        # Options: range, roundrobin, sticky, cooperative-sticky
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
        
        # For cooperative rebalancing (less disruptive)
        # Consumers keep processing unaffected partitions during rebalance
        
    listener:
      # Number of consumer threads
      concurrency: 4
      
      # Acknowledgment mode
      ack-mode: manual_immediate
```

### Partition Assignment Strategies

```java
package com.systemdesign.messaging.consumergroups;

/**
 * Explanation of partition assignment strategies.
 */
public class PartitionAssignmentStrategies {
    
    /*
     * RANGE ASSIGNOR (default)
     * ========================
     * Assigns partitions in ranges to consumers.
     * 
     * Example: 6 partitions, 2 consumers
     * Consumer A: P0, P1, P2
     * Consumer B: P3, P4, P5
     * 
     * Pros: Simple, predictable
     * Cons: Can be uneven with multiple topics
     */
    
    /*
     * ROUND ROBIN ASSIGNOR
     * ====================
     * Assigns partitions in round-robin fashion.
     * 
     * Example: 6 partitions, 2 consumers
     * Consumer A: P0, P2, P4
     * Consumer B: P1, P3, P5
     * 
     * Pros: Even distribution across topics
     * Cons: More partition movement during rebalance
     */
    
    /*
     * STICKY ASSIGNOR
     * ===============
     * Like round-robin but tries to minimize partition movement.
     * 
     * Before: Consumer A (P0, P2, P4), Consumer B (P1, P3, P5)
     * Consumer C joins:
     * After: Consumer A (P0, P2), Consumer B (P1, P3), Consumer C (P4, P5)
     * 
     * Pros: Minimizes rebalance impact
     * Cons: Slightly more complex
     */
    
    /*
     * COOPERATIVE STICKY ASSIGNOR (recommended)
     * =========================================
     * Incremental rebalancing - consumers don't stop processing
     * unaffected partitions during rebalance.
     * 
     * Traditional rebalance: ALL consumers stop, reassign, resume
     * Cooperative rebalance: Only affected partitions stop
     * 
     * Pros: Less disruption, faster rebalances
     * Cons: Requires all consumers to use same strategy
     */
}
```

### Application Configuration

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    
    consumer:
      group-id: order-processor
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      
      properties:
        spring.json.trusted.packages: "com.systemdesign.messaging"
        
        # Consumer group settings
        session.timeout.ms: 10000
        heartbeat.interval.ms: 3000
        max.poll.interval.ms: 300000
        max.poll.records: 500
        
        # Rebalancing
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
    
    listener:
      concurrency: 4
      ack-mode: manual_immediate
      
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

#### 1. More Consumers Than Partitions

**Problem:**
```
Topic: orders (4 partitions)
Consumer Group: order-processor (6 consumers)

Assignment:
Consumer A: P0
Consumer B: P1
Consumer C: P2
Consumer D: P3
Consumer E: (idle!)
Consumer F: (idle!)

Two consumers are doing nothing!
```

**Rule:** Number of consumers ≤ Number of partitions

**Solution:** Either reduce consumers or increase partitions.

#### 2. Not Handling Rebalances

**Problem:**
```java
// Consumer has in-memory state
Map<String, Integer> counts = new HashMap<>();

@KafkaListener(topics = "events")
public void process(Event event) {
    counts.merge(event.getType(), 1, Integer::sum);
}

// During rebalance, this consumer might lose partitions
// but still has counts from those partitions!
// State is now incorrect.
```

**Solution:**
```java
@Override
public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
    // Flush counts for revoked partitions
    for (TopicPartition p : partitions) {
        flushCountsForPartition(p);
    }
}
```

#### 3. Long Processing Time Causing Rebalance

**Problem:**
```
max.poll.interval.ms = 300000 (5 minutes)

Consumer polls 500 records
Each record takes 1 second to process
Total: 500 seconds = 8.3 minutes

Consumer exceeds max.poll.interval!
Broker thinks consumer is dead.
Triggers unnecessary rebalance.
```

**Solution:**
- Reduce `max.poll.records`
- Increase `max.poll.interval.ms`
- Process faster (async, batch)

#### 4. Committing Before Processing

**Wrong:**
```java
@KafkaListener(topics = "orders")
public void process(Order order, Acknowledgment ack) {
    ack.acknowledge();  // Commit first!
    processOrder(order);  // What if this fails?
}
// If processing fails, message is lost!
```

**Right:**
```java
@KafkaListener(topics = "orders")
public void process(Order order, Acknowledgment ack) {
    processOrder(order);  // Process first
    ack.acknowledge();    // Then commit
}
```

### Performance Considerations

**Partition Count:**
- More partitions = more parallelism
- But: more overhead, more files, longer leader election
- Rule of thumb: Start with 3-6x expected consumer count

**Consumer Count:**
- Match partition count for optimal parallelism
- Fewer consumers = some handle multiple partitions
- More consumers = some sit idle

**Poll Settings:**
```
max.poll.records: Balance between throughput and latency
- High (1000): Better throughput, higher latency
- Low (100): Lower throughput, lower latency

fetch.min.bytes: Minimum data to fetch
- High: Better batching, higher latency
- Low: Lower latency, more requests
```

---

## 8️⃣ When NOT to Use This

### When Consumer Groups Are Unnecessary

1. **Single consumer**: If you only need one consumer, consumer groups add complexity.

2. **Broadcast to all**: If every consumer needs every message, use separate consumer groups (one per consumer) or Pub/Sub.

3. **Stateless processing**: If processing doesn't need coordination, simpler patterns might work.

### When Consumer Groups Are Problematic

1. **Ordering across partitions**: Consumer groups maintain order within partitions, not across them. If you need global ordering, use single partition (limits scalability).

2. **Exactly-once across consumers**: Consumer groups provide at-least-once. For exactly-once, need additional idempotency.

3. **Dynamic partition count**: Changing partition count triggers rebalance and can cause issues.

### Anti-Patterns

1. **One partition per message type**: Creates too many partitions, limits parallelism.

2. **Consumer group per consumer**: Defeats the purpose, each consumer gets all messages.

3. **Ignoring rebalance events**: Leads to state inconsistency.

---

## 9️⃣ Comparison with Alternatives

### Consumer Groups vs Message Queues

| Aspect | Kafka Consumer Groups | Traditional Queue (RabbitMQ) |
|--------|----------------------|------------------------------|
| **Ordering** | Per-partition | Per-queue |
| **Replay** | Yes (seek to offset) | No (message deleted) |
| **Scaling** | Add consumers ≤ partitions | Add consumers freely |
| **State** | Broker tracks offsets | Broker tracks delivery |
| **Message retention** | Time/size based | Until consumed |

### Consumer Groups vs Pub/Sub

| Aspect | Consumer Groups | Pub/Sub |
|--------|----------------|---------|
| **Delivery** | One consumer per partition | All subscribers |
| **Use case** | Work distribution | Event notification |
| **Scaling** | Add consumers to group | Add subscribers |
| **Coordination** | Automatic | None needed |

### Assignment Strategy Comparison

| Strategy | Pros | Cons | Use When |
|----------|------|------|----------|
| Range | Simple, predictable | Uneven with multiple topics | Single topic |
| Round Robin | Even distribution | More movement on rebalance | Multiple topics |
| Sticky | Minimizes movement | Slightly complex | General use |
| Cooperative Sticky | Non-blocking rebalance | Requires all consumers | Production (recommended) |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q1: What is a consumer group?**

**Answer:**
A consumer group is a set of consumers that work together to consume messages from a topic. The key feature is that each message is delivered to only one consumer in the group.

Think of it like a team of workers sharing a task list. If there are 100 tasks and 5 workers, each worker gets about 20 tasks. No task is done twice, and all tasks get done.

In Kafka, partitions are assigned to consumers in the group. Each partition goes to exactly one consumer, but one consumer can handle multiple partitions.

**Q2: What happens when a consumer in a group fails?**

**Answer:**
When a consumer fails (stops sending heartbeats), the broker detects this and triggers a "rebalance." During rebalance:

1. The failed consumer's partitions are redistributed to remaining consumers
2. Other consumers in the group take over those partitions
3. Processing continues from the last committed offset

For example, if Consumer A was handling partitions 0 and 1, and it fails, Consumer B might take over partition 0 and Consumer C might take over partition 1. No messages are lost because the offset (position) was tracked.

### L5 (Senior) Questions

**Q3: How would you handle a scenario where message processing takes longer than max.poll.interval.ms?**

**Answer:**
This is a common issue. Options:

1. **Increase max.poll.interval.ms**: If processing legitimately takes long, increase the timeout. But this delays failure detection.

2. **Reduce max.poll.records**: Fetch fewer messages per poll so processing completes faster.

3. **Async processing**: 
   - Poll messages
   - Submit to thread pool
   - Poll again immediately (heartbeat continues)
   - Track which messages are processed
   - Commit when batch is done

4. **Pause and resume**:
   ```java
   consumer.pause(partitions);  // Stop fetching
   // Process messages
   consumer.resume(partitions);  // Resume fetching
   ```

5. **Separate heartbeat thread** (Kafka 0.10.1+): Heartbeats are sent by a background thread, so long processing doesn't affect heartbeat.

**Q4: How do you ensure exactly-once processing with consumer groups?**

**Answer:**
Consumer groups provide at-least-once by default. For exactly-once:

1. **Idempotent processing**: Design consumers so processing the same message twice has no effect. Use unique IDs, check before insert, etc.

2. **Transactional processing**:
   - Read message
   - Begin database transaction
   - Process and write to database
   - Commit offset to Kafka (in same transaction if possible)
   - Commit database transaction

3. **Kafka transactions** (for Kafka-to-Kafka):
   - Use `read_committed` isolation
   - Process and produce in transaction
   - Offset committed atomically with output

4. **External offset storage**:
   - Store offset in same database as processed data
   - On restart, read offset from database
   - Atomic: either both update or neither

### L6 (Staff) Questions

**Q5: Design a consumer group architecture for processing 1 million events per second.**

**Answer:**
At 1M events/sec, we need careful architecture:

**Partition design:**
- Single consumer processes ~10K events/sec (assumption)
- Need ~100 consumers
- Need at least 100 partitions (use 128 for headroom)
- Partition by event key for ordering

**Consumer deployment:**
- Deploy 100+ consumer instances
- Use Kubernetes with HPA for auto-scaling
- Monitor consumer lag, scale up if lag grows

**Processing optimization:**
- Batch processing (process 1000 events at once)
- Async I/O for database writes
- Connection pooling
- Local caching

**Failure handling:**
- Cooperative sticky assignor (minimize rebalance impact)
- Short session timeout (detect failures quickly)
- DLQ for poison messages

**Monitoring:**
- Consumer lag per partition
- Processing latency percentiles
- Rebalance frequency
- Error rates

**Q6: How would you handle a scenario where one partition has much more data than others (hot partition)?**

**Answer:**
Hot partitions are a common problem. Solutions:

1. **Better partitioning key**: If using user_id and one user has 50% of traffic, partition by something else (event_type + user_id hash).

2. **Salting**: Add random suffix to partition key
   ```
   key = userId + "_" + random(0, 10)
   ```
   Distributes one user's events across 10 partitions.
   Downside: Lose ordering for that user.

3. **Dedicated partition**: Give the hot key its own partition and dedicated consumer.

4. **Rate limiting**: Limit events from hot sources at ingestion.

5. **Sampling**: For analytics, sample hot partition instead of processing all.

6. **Separate topic**: Route hot traffic to dedicated topic with more partitions.

The right solution depends on whether you need ordering for the hot key and how temporary the hotness is.

---

## 1️⃣1️⃣ One Clean Mental Summary

Consumer groups enable parallel processing of messages from a topic. All consumers with the same `group.id` share the partitions, with each partition assigned to exactly one consumer. This provides automatic load balancing (work distributed across consumers), fault tolerance (if a consumer dies, its partitions are reassigned), and scalability (add consumers up to partition count). Different consumer groups process the same messages independently, enabling multiple services to consume the same event stream. Key concepts: partitions determine parallelism limit, offsets track progress per group, rebalancing redistributes partitions when consumers join/leave. The formula: consumers in group ≤ partitions for optimal parallelism.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│              CONSUMER GROUPS CHEAT SHEET                     │
├─────────────────────────────────────────────────────────────┤
│ CORE CONCEPT                                                 │
│   Same group.id = share partitions (work distribution)      │
│   Different group.id = each gets all messages (pub/sub)     │
├─────────────────────────────────────────────────────────────┤
│ KEY RULES                                                    │
│   • Each partition → exactly one consumer in group          │
│   • Each consumer → can handle multiple partitions          │
│   • Consumers in group ≤ partitions (extras sit idle)       │
│   • Messages in partition → processed in order              │
├─────────────────────────────────────────────────────────────┤
│ REBALANCING TRIGGERS                                         │
│   • Consumer joins group                                    │
│   • Consumer leaves (graceful)                              │
│   • Consumer fails (heartbeat timeout)                      │
│   • Partitions added to topic                               │
├─────────────────────────────────────────────────────────────┤
│ KEY SETTINGS                                                 │
│   group.id: Consumer group identifier                       │
│   session.timeout.ms: Time before consumer considered dead  │
│   heartbeat.interval.ms: Heartbeat frequency                │
│   max.poll.interval.ms: Max time between polls              │
│   max.poll.records: Records per poll                        │
├─────────────────────────────────────────────────────────────┤
│ ASSIGNMENT STRATEGIES                                        │
│   Range: Contiguous partitions per consumer                 │
│   RoundRobin: Distributed evenly                            │
│   Sticky: Minimize movement on rebalance                    │
│   CooperativeSticky: Non-blocking rebalance (recommended)   │
├─────────────────────────────────────────────────────────────┤
│ OFFSET COMMIT                                                │
│   Auto: Commit periodically (risk: reprocess on crash)      │
│   Manual Sync: Commit and wait (slower, safer)              │
│   Manual Async: Commit without waiting (faster, riskier)    │
├─────────────────────────────────────────────────────────────┤
│ COMMON MISTAKES                                              │
│   ✗ More consumers than partitions (some idle)              │
│   ✗ Not handling rebalance events                           │
│   ✗ Long processing exceeding max.poll.interval             │
│   ✗ Committing before processing                            │
└─────────────────────────────────────────────────────────────┘
```

