# Ticket Booking System - Production Deep Dives (Core)

## Overview

This document covers core production components: asynchronous messaging (Kafka for booking events), caching strategy (Redis for seat locks and availability), and distributed locking for seat management.

---

## 1. Async, Messaging & Event Flow (Kafka)

### A) CONCEPT: What is Kafka in Ticket Booking Context?

Kafka serves as the event backbone for:
1. **Booking Events**: Stream booking lifecycle (created, confirmed, cancelled)
2. **Seat Events**: Stream seat lock/unlock events
3. **Payment Events**: Stream payment processing events
4. **Notification Events**: Trigger email/SMS for tickets

### B) OUR USAGE: How We Use Kafka Here

**Topic Design:**

| Topic | Purpose | Partitions | Key |
|-------|---------|------------|-----|
| `booking-events` | Booking lifecycle | 10 | booking_id |
| `seat-events` | Seat lock/unlock | 5 | show_id |
| `payment-events` | Payment processing | 5 | payment_id |

**Producer Configuration:**

```java
@Configuration
public class BookingKafkaConfig {
    
    @Bean
    public ProducerFactory<String, BookingEvent> bookingEventProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Normal Flow: Booking Creation Event**

```
Step 1: Booking Created
Booking Service: Creates booking 'booking_123456'
  │
  ▼
Step 2: Publish Event
Kafka Producer: order-events, partition 3, key 'booking_123456'
  Message: {"event_type": "booking.created", "booking_id": "booking_123456"}
  │
  ▼
Step 3: Consumers Process
- Notification Service: Send ticket email
- Analytics Service: Update booking metrics
- Seat Service: Confirm seat locks
```

---

## 2. Caching (Redis)

### A) CONCEPT: What is Redis Caching in Booking Context?

Redis provides:
1. **Seat Lock Cache**: Fast seat locking/unlocking
2. **Seat Availability Cache**: Real-time seat map
3. **Booking Cache**: Recent bookings for quick access

### B) OUR USAGE: How We Use Redis Here

**Cache Types:**

| Cache | Key | TTL | Purpose |
|-------|-----|-----|---------|
| Seat Lock | `seat_lock:{show_id}:{seat_id}` | 5 minutes | Active seat locks |
| Seat Map | `seat_map:{show_id}` | 1 hour | Seat availability |
| Booking | `booking:{booking_id}` | 24 hours | Recent bookings |

**Seat Locking:**

```java
@Service
public class SeatLockService {
    
    public LockResult lockSeat(String showId, String seatId, String userId) {
        String key = "seat_lock:" + showId + ":" + seatId;
        
        // Try to acquire lock (atomic operation)
        Boolean acquired = redis.opsForValue().setIfAbsent(
            key, 
            userId, 
            Duration.ofSeconds(300)  // 5 minutes
        );
        
        if (acquired) {
            return LockResult.success();
        } else {
            return LockResult.failed("Seat already locked");
        }
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Normal Flow: Seat Locking**

```
Step 1: User Selects Seat
User → Seat Service: Lock seat_1 for show_123
  │
  ▼
Step 2: Check Redis Lock
Redis: GET seat_lock:show_123:seat_1
Result: nil (available)
  │
  ▼
Step 3: Acquire Lock
Redis: SET seat_lock:show_123:seat_1 user_789 EX 300
Result: OK (lock acquired)
  │
  ▼
Step 4: Store Lock Record
PostgreSQL: INSERT INTO seat_locks
  │
  ▼
Success: Seat Locked
User receives lock_id, 5-minute timer starts
```

**Failure Flow: Seat Already Locked**

```
Step 1: User Tries to Lock
User → Seat Service: Lock seat_1
  │
  ▼
Step 2: Check Redis Lock
Redis: GET seat_lock:show_123:seat_1
Result: "user_456" (already locked)
  │
  ▼
Step 3: Return Error
Seat Service → User: "Seat already locked by another user"
```

---

## 3. Distributed Locking for Seat Management

### A) CONCEPT: What is Distributed Locking?

Distributed locking ensures only one process can lock a resource at a time across multiple servers.

**Why Needed:**
- Multiple booking servers handling requests
- Race condition: Two users selecting same seat simultaneously
- Must prevent double booking

### B) OUR USAGE: How We Use Distributed Locking

**Implementation:**

```java
@Service
public class SeatLockService {
    
    private final RedisTemplate<String, String> redis;
    private final DistributedLock distributedLock;
    
    public LockResult lockSeats(String showId, List<String> seatIds, String userId) {
        // Acquire distributed lock for entire operation
        String lockKey = "distributed_lock:show:" + showId;
        Lock lock = distributedLock.acquire(lockKey, Duration.ofSeconds(5));
        
        try {
            // Lock all seats atomically
            for (String seatId : seatIds) {
                String seatKey = "seat_lock:" + showId + ":" + seatId;
                Boolean acquired = redis.opsForValue().setIfAbsent(
                    seatKey, userId, Duration.ofSeconds(300)
                );
                
                if (!acquired) {
                    // Release already acquired locks
                    releaseSeats(showId, seatIds.subList(0, seatIds.indexOf(seatId)));
                    return LockResult.failed("Seat " + seatId + " unavailable");
                }
            }
            
            return LockResult.success();
            
        } finally {
            lock.release();
        }
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Concurrent Booking Scenario:**

```
Time T+0ms: User A selects seat_1
  Seat Service A: Acquires distributed lock
  Redis: SET seat_lock:show_123:seat_1 user_A EX 300
  Result: OK (lock acquired)

Time T+10ms: User B selects seat_1
  Seat Service B: Tries to acquire distributed lock
  Lock: Already held by Service A, waits...
  
Time T+50ms: Service A completes, releases lock
  Service A: Releases distributed lock
  
Time T+60ms: Service B acquires lock
  Redis: GET seat_lock:show_123:seat_1
  Result: "user_A" (already locked)
  Service B: Returns error "Seat unavailable"
```

---

## Summary

| Component | Technology | Key Configuration |
|-----------|------------|-------------------|
| Event Streaming | Kafka | 10 partitions, booking_id key |
| Seat Lock Cache | Redis | 5-minute TTL |
| Seat Map Cache | Redis | 1-hour TTL |
| Distributed Locking | Redis + Custom | Prevents race conditions |
| Lock Timeout | 5 minutes | Prevents seat hoarding |

