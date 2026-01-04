# Ticket Booking System - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, failure scenarios, and level-specific expectations.

---

## Trade-off Questions

### Q1: How do you prevent double booking?

**Answer:**

**Problem:** Multiple users trying to book the same seat simultaneously

**Solution: Distributed Locking + Database Constraints**

```java
@Service
public class SeatLockService {
    
    public LockResult lockSeat(String showId, String seatId, String userId) {
        // 1. Acquire distributed lock
        String lockKey = "distributed_lock:show:" + showId;
        Lock lock = distributedLock.acquire(lockKey, Duration.ofSeconds(5));
        
        try {
            // 2. Check and lock seat atomically
            String seatKey = "seat_lock:" + showId + ":" + seatId;
            Boolean acquired = redis.opsForValue().setIfAbsent(
                seatKey, userId, Duration.ofSeconds(300)
            );
            
            if (!acquired) {
                return LockResult.failed("Seat already locked");
            }
            
            // 3. Store lock record in database
            seatLockRepository.save(new SeatLock(showId, seatId, userId));
            
            return LockResult.success();
            
        } finally {
            lock.release();
        }
    }
}
```

**Additional Safeguards:**
- Database unique constraint on (show_id, seat_id) where status = 'locked'
- Optimistic locking (version field)
- Lock expiration (5 minutes)

---

### Q2: What happens if seat lock expires during payment?

**Answer:**

**Problem:** User locks seat, payment takes 6 minutes, lock expires

**Solution: Extend Lock on Payment Start**

```java
@Service
public class BookingService {
    
    public BookingResult createBooking(CreateBookingRequest request) {
        // 1. Validate lock still exists
        SeatLock lock = seatLockService.getLock(request.getLockId());
        if (lock == null || lock.isExpired()) {
            return BookingResult.failed("Seat lock expired");
        }
        
        // 2. Extend lock before payment
        seatLockService.extendLock(request.getLockId(), Duration.ofMinutes(10));
        
        // 3. Process payment
        PaymentResult payment = paymentService.processPayment(request.getPayment());
        
        if (!payment.isSuccess()) {
            // Release lock on payment failure
            seatLockService.releaseLock(request.getLockId());
            return BookingResult.failed("Payment failed");
        }
        
        // 4. Create booking (locks are confirmed)
        Order order = orderService.createOrder(request, payment.getPaymentId());
        
        return BookingResult.success(order);
    }
}
```

---

### Q3: How do you handle seat lock cleanup?

**Answer:**

**Problem:** Expired locks need to be cleaned up

**Solution: Background Job + TTL**

```java
@Component
public class SeatLockCleanupJob {
    
    @Scheduled(fixedRate = 60000)  // Every minute
    public void cleanupExpiredLocks() {
        // 1. Find expired locks in database
        List<SeatLock> expiredLocks = seatLockRepository.findExpiredLocks();
        
        for (SeatLock lock : expiredLocks) {
            // 2. Release lock in Redis
            redis.delete("seat_lock:" + lock.getShowId() + ":" + lock.getSeatId());
            
            // 3. Update lock status in database
            lock.setStatus("expired");
            seatLockRepository.save(lock);
        }
    }
}
```

**Redis TTL:**
- Locks automatically expire in Redis (5 minutes)
- Background job cleans up database records
- Prevents orphaned locks

---

## Scaling Questions

### Q4: How would you scale from 1M to 10M bookings/day?

**Answer:**

**Current State (1M bookings/day):**
- 42 servers
- 1,000 bookings/second peak

**Scaling to 10M bookings/day:**

1. **Horizontal Scaling**
   ```
   Servers: 42 → 400 (10x)
   ```

2. **Database Sharding**
   ```
   Bookings table: Shard by show_id hash
   10 shards → 100 shards
   ```

3. **Redis Cluster Scaling**
   ```
   More Redis nodes for seat locks
   Partition locks by show_id
   ```

4. **Event Streaming**
   ```
   Kafka: 10 partitions → 100 partitions
   ```

**Bottlenecks:**
- Seat locking contention (partition by show_id)
- Database write capacity (sharding)
- Payment gateway rate limits (multiple gateways)

---

## Failure Scenarios

### Scenario 1: Seat Lock Service Down

**Problem:** Cannot lock seats, bookings fail

**Solution:**
1. Circuit breaker opens
2. Reject seat selection (better than inconsistent state)
3. Show user-friendly error
4. Retry when service recovers

**Trade-off:** Availability vs. Consistency
- Choose consistency (reject) over availability
- Cannot risk double booking

---

### Scenario 2: Payment Succeeds but Booking Creation Fails

**Problem:** Payment charged but booking not created

**Solution: Saga Compensation**

```java
@Service
public class BookingSagaOrchestrator {
    
    public BookingResult createBooking(CreateBookingRequest request) {
        String lockId = request.getLockId();
        String paymentId = null;
        
        try {
            // Process payment
            PaymentResult payment = paymentService.processPayment(request.getPayment());
            paymentId = payment.getPaymentId();
            
            // Create booking
            Order order = orderService.createOrder(request, paymentId);
            
            return BookingResult.success(order);
            
        } catch (OrderCreationException e) {
            // Compensation: Refund payment, release locks
            if (paymentId != null) {
                paymentService.refund(paymentId);
            }
            seatLockService.releaseLock(lockId);
            
            return BookingResult.failed("Booking creation failed");
        }
    }
}
```

---

## Level-Specific Expectations

### L4 (Entry-Level)

**What's Expected:**
- Basic booking flow understanding
- Simple seat locking concept
- Basic error handling

---

### L5 (Mid-Level)

**What's Expected:**
- Distributed locking understanding
- Double booking prevention
- Saga pattern for compensation

---

### L6 (Senior)

**What's Expected:**
- System evolution (MVP → Production → Scale)
- Trade-off analysis
- Edge cases (lock expiration, concurrent bookings)
- Cost optimization

---

## Common Interviewer Pushbacks

### "Your seat locking is too complex"

**Response:**
"For MVP, we could use database locks. As we scale, distributed locking becomes necessary. The key is to prevent double booking while maintaining performance."

---

### "What if Redis is down?"

**Response:**
"We'd fall back to database-only locking (slower but works). We'd also use circuit breakers to fail fast and show users a clear error message."

---

## Summary

| Question Type | Key Points to Cover |
|--------------|---------------------|
| Trade-offs | Distributed locking vs database locks |
| Scaling | Horizontal scaling, database sharding |
| Failures | Compensation, retry logic, circuit breakers |
| Edge cases | Lock expiration, concurrent bookings |
| Evolution | MVP → Production → Scale phases |

