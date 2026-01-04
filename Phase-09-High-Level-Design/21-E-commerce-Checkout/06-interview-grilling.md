# E-commerce Checkout - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, failure scenarios, and level-specific expectations for an e-commerce checkout system design.

---

## Trade-off Questions

### Q1: Why use Saga pattern instead of two-phase commit (2PC)?

**Answer:**

**Two-Phase Commit (2PC):**
- Blocking protocol (all participants wait)
- Doesn't scale well (coordination overhead)
- Single point of failure (coordinator)
- Long-lived locks

**Saga Pattern:**
- Non-blocking (each service commits independently)
- Scales horizontally
- No single coordinator
- Compensation handles failures

**Trade-off Analysis:**

| Factor | 2PC | Saga Pattern |
|--------|-----|--------------|
| Blocking | Yes | No |
| Scalability | Poor | Excellent |
| Failure Handling | Rollback | Compensation |
| Complexity | Medium | High (compensation logic) |

**Key insight:** Saga pattern is better for distributed systems at scale, even though it requires more complex compensation logic.

---

### Q2: How do you prevent inventory overselling?

**Answer:**

**Problem:** Multiple users trying to buy the last item simultaneously

**Solution: Distributed Locking + Reservation**

```java
@Service
public class InventoryService {
    
    public ReservationResult reserveInventory(String productId, int quantity) {
        // 1. Acquire distributed lock
        String lockKey = "lock:inventory:" + productId;
        Lock lock = distributedLock.acquire(lockKey, Duration.ofSeconds(5));
        
        try {
            // 2. Check availability (within lock)
            Product product = productRepository.findById(productId);
            int available = product.getStockQuantity() - product.getReservedQuantity();
            
            if (available < quantity) {
                return ReservationResult.failed("Insufficient inventory");
            }
            
            // 3. Create reservation
            Reservation reservation = Reservation.builder()
                .productId(productId)
                .quantity(quantity)
                .expiresAt(Instant.now().plusSeconds(1800))  // 30 minutes
                .build();
            
            reservationRepository.save(reservation);
            
            // 4. Update reserved quantity
            product.setReservedQuantity(product.getReservedQuantity() + quantity);
            productRepository.save(product);
            
            return ReservationResult.success(reservation);
            
        } finally {
            lock.release();
        }
    }
}
```

**Additional Safeguards:**
- Database constraints (CHECK available_quantity >= 0)
- Optimistic locking (version field)
- Reservation expiration (release after timeout)

---

### Q3: How do you handle payment idempotency?

**Answer:**

**Problem:** User clicks "Pay" multiple times, payment processed multiple times

**Solution: Idempotency Key + Database Constraint**

```java
@Service
public class PaymentService {
    
    public PaymentResult processPayment(PaymentRequest request, String idempotencyKey) {
        // 1. Check if payment already processed
        if (idempotencyKey != null) {
            Payment existing = paymentRepository.findByIdempotencyKey(idempotencyKey);
            if (existing != null) {
                return PaymentResult.from(existing);  // Return existing result
            }
        }
        
        // 2. Process payment with gateway
        GatewayResponse gatewayResponse = paymentGateway.charge(
            request.getAmount(),
            request.getPaymentMethod()
        );
        
        // 3. Store payment (idempotency key is unique constraint)
        Payment payment = Payment.builder()
            .idempotencyKey(idempotencyKey)
            .amount(request.getAmount())
            .status(gatewayResponse.isSuccess() ? "succeeded" : "failed")
            .gatewayTransactionId(gatewayResponse.getTransactionId())
            .build();
        
        try {
            paymentRepository.save(payment);
        } catch (DataIntegrityViolationException e) {
            // Idempotency key already exists, return existing payment
            Payment existing = paymentRepository.findByIdempotencyKey(idempotencyKey);
            return PaymentResult.from(existing);
        }
        
        return PaymentResult.from(payment);
    }
}
```

**Database Constraint:**
```sql
CREATE UNIQUE INDEX idx_payments_idempotency ON payments(idempotency_key);
```

---

### Q4: What happens if payment succeeds but order creation fails?

**Answer:**

**Problem:** Payment charged but order not created (inconsistent state)

**Solution: Saga Compensation + Retry**

```java
@Service
public class OrderSagaOrchestrator {
    
    public OrderResult createOrder(CreateOrderRequest request) {
        String sagaId = UUID.randomUUID().toString();
        String reservationId = null;
        String paymentId = null;
        
        try {
            // Step 1: Reserve inventory
            ReservationResult reservation = inventoryService.reserveInventory(
                request.getItems(), sagaId
            );
            reservationId = reservation.getReservationId();
            
            // Step 2: Process payment
            PaymentResult payment = paymentService.processPayment(
                request.getPaymentMethod(), request.getAmount(), sagaId
            );
            paymentId = payment.getPaymentId();
            
            // Step 3: Create order
            Order order = orderService.createOrder(
                request, reservationId, paymentId, sagaId
            );
            
            // Success
            return OrderResult.success(order);
            
        } catch (OrderCreationException e) {
            // Compensation: Refund payment and release inventory
            if (paymentId != null) {
                paymentService.refund(paymentId);
            }
            if (reservationId != null) {
                inventoryService.releaseReservation(reservationId);
            }
            
            // Retry order creation (with existing payment)
            return retryOrderCreation(request, reservationId, paymentId);
        }
    }
}
```

**Additional Safeguards:**
- Background job to detect orphaned payments
- Automatic refund for payments without orders
- Alerting for compensation failures

---

## Scaling Questions

### Q5: How would you scale from 10M to 100M orders/day?

**Answer:**

**Current State (10M orders/day):**
- 41 servers
- 1,000 orders/second peak
- PostgreSQL single primary

**Scaling to 100M orders/day:**

1. **Horizontal Scaling**
   ```
   Servers: 41 → 400 (10x)
   Services scale proportionally
   ```

2. **Database Sharding**
   ```
   Orders table: Shard by user_id hash
   10 shards → 100 shards
   ```

3. **Payment Gateway Scaling**
   ```
   Multiple payment gateways (load balancing)
   Queue payments for peak handling
   ```

4. **Caching**
   ```
   More Redis nodes
   Increase cache hit rates
   ```

5. **Event Streaming**
   ```
   Kafka: 10 partitions → 100 partitions
   More Kafka brokers
   ```

**Bottlenecks:**
- Payment gateway rate limits (use multiple gateways)
- Database write capacity (sharding)
- Inventory locking contention (optimistic locking)

---

## Failure Scenarios

### Scenario 1: Payment Gateway Down

**Problem:** Payment gateway is unavailable, users can't checkout

**Solution:**
1. Circuit breaker opens after 50% failure rate
2. Queue payments for retry
3. Show user-friendly message
4. Retry payments when gateway recovers

**Implementation:**
```java
@Service
public class PaymentService {
    
    private final CircuitBreaker circuitBreaker;
    private final KafkaTemplate<String, PaymentRequest> kafka;
    
    public PaymentResult processPayment(PaymentRequest request) {
        if (circuitBreaker.getState() == CircuitBreaker.State.OPEN) {
            // Queue for retry
            kafka.send("payment-retry-queue", request);
            return PaymentResult.queued();
        }
        
        try {
            return paymentGateway.charge(request);
        } catch (Exception e) {
            circuitBreaker.recordFailure();
            kafka.send("payment-retry-queue", request);
            return PaymentResult.queued();
        }
    }
}
```

---

### Scenario 2: Inventory Service Down

**Problem:** Can't check inventory, can't reserve items

**Solution:**
1. Circuit breaker opens
2. Reject checkout (better than overselling)
3. Show user-friendly error
4. Retry when service recovers

**Trade-off:** Availability vs. Consistency
- Choose consistency (reject checkout) over availability
- Cannot risk overselling inventory

---

## Level-Specific Expectations

### L4 (Entry-Level)

**What's Expected:**
- Basic checkout flow understanding
- Simple database design
- Basic error handling

**Sample Answer:**
> "The checkout system would have a cart, payment processing, and order creation. Users add items to cart, enter payment info, and we create an order. We'd use a database to store orders and a payment gateway for processing payments."

---

### L5 (Mid-Level)

**What's Expected:**
- Saga pattern understanding
- Inventory reservation logic
- Payment idempotency
- Failure handling

**Sample Answer:**
> "I'd use a Saga pattern for order creation: reserve inventory, process payment, create order. If any step fails, we compensate. For inventory, I'd use distributed locking to prevent overselling. Payments need idempotency keys to prevent duplicate charges."

---

### L6 (Senior)

**What's Expected:**
- System evolution (MVP → Production → Scale)
- Trade-off analysis
- Edge cases and failure scenarios
- Cost optimization

**Sample Answer:**
> "For MVP, a single service with a database works. As we scale, we split into services and use Saga pattern. The hardest problems are inventory consistency (distributed locking) and payment idempotency (idempotency keys). At 100M orders/day, we'd shard the database and use multiple payment gateways."

---

## Common Interviewer Pushbacks

### "Your system is too complex for MVP"

**Response:**
"You're right. For MVP, I'd start with a single service and database. As we scale, we'd split into services and add Saga pattern. The key is to design for evolution, not over-engineer initially."

---

### "What if payment gateway is slow?"

**Response:**
"We'd set timeouts (10 seconds) and show users a processing message. If payment takes longer, we'd queue it for async processing and notify the user when complete. We'd also use circuit breakers to fail fast if the gateway is down."

---

## Summary

| Question Type | Key Points to Cover |
|--------------|---------------------|
| Trade-offs | Saga vs 2PC, consistency vs availability |
| Scaling | Horizontal scaling, database sharding |
| Failures | Compensation, retry logic, circuit breakers |
| Edge cases | Inventory overselling, payment idempotency |
| Evolution | MVP → Production → Scale phases |


