# Idempotency Deep Dive

## 0️⃣ Prerequisites

Before diving into this deep dive on idempotency, you should understand:

- **Idempotency Basics** (covered in Phase 1, Topic 13): The fundamental concept that performing an operation multiple times has the same effect as performing it once
- **Distributed Systems** (covered in Phase 1, Topic 01): Systems where multiple computers communicate over unreliable networks
- **Failure Modes** (covered in Phase 1, Topic 07): Network failures, timeouts, and retries
- **Consistency Models** (covered in Phase 1, Topic 12): How different systems guarantee data consistency

**Quick refresher from Phase 1**: Idempotency means `f(f(x)) = f(x)`. In distributed systems, this is critical because network failures cause retries. Without idempotency, a retry might charge a customer twice or create duplicate orders. Phase 1 covered the basics. This topic goes deeper into advanced patterns, edge cases, and production implementations.

---

## 1️⃣ What Problem Does This Exist to Solve?

### Beyond the Basics: The Deep Problems

Phase 1 covered the fundamental duplicate request problem. But in production distributed systems, idempotency challenges are more nuanced:

**Problem 1: Partial Failures**
```
Client sends: "Transfer $100 from A to B"

Server:
  1. Debit account A: $100 ✓
  2. Credit account B: $100 (server crashes!)

Client retries with same idempotency key:
  - Server sees key, returns "success" (cached from step 1)
  - But account B was never credited!
```

**Problem 2: Idempotency Across Services**
```
Order Service                     Payment Service
     │                                  │
     │ "Process order 123"              │
     ├─────────────────────────────────>│
     │                                  │ Charge card ✓
     │                                  │ Response lost!
     │ (timeout, retry)                 │
     ├─────────────────────────────────>│
     │                                  │ Already processed...
     │                                  │ But Order Service
     │                                  │ doesn't know!
```

**Problem 3: Idempotency Key Collisions**
```
User A: POST /orders, Idempotency-Key: "abc123"
User B: POST /orders, Idempotency-Key: "abc123" (same key!)

Without proper scoping, User B might get User A's response!
```

**Problem 4: Time-Based Idempotency**
```
Request at 10:00:00: "Set price to $100"
Request at 10:00:01: "Set price to $150"
Request at 10:00:00 (retry, arrived late): "Set price to $100"

Should the retry overwrite the newer price?
```

### What Breaks in Production Without Advanced Idempotency

- **Financial systems**: Money appears or disappears
- **Inventory systems**: Stock counts become incorrect
- **Order systems**: Duplicate orders, partial orders
- **Messaging systems**: Messages delivered multiple times
- **Workflow systems**: Steps executed multiple times
- **Distributed transactions**: Inconsistent state across services

### Real-World Production Incidents

**Stripe's Idempotency Evolution**:
Stripe's idempotency implementation has evolved significantly. Early versions only stored the response. They later added:
- Request body fingerprinting (reject different body with same key)
- Scoping by API key
- Handling in-flight requests
- Detailed error responses for idempotency violations

**AWS Lambda's At-Least-Once Delivery**:
AWS Lambda guarantees at-least-once execution, not exactly-once. This means your Lambda function might run multiple times for the same event. AWS explicitly recommends making Lambda functions idempotent.

**Uber's Payment Idempotency**:
Uber processes millions of payments daily. They've built sophisticated idempotency handling that includes:
- Idempotency at the API gateway level
- Idempotency at the service level
- Database-level constraints
- Reconciliation systems for edge cases

---

## 2️⃣ Intuition and Mental Model

### The Bank Teller Analogy (Extended)

In Phase 1, we used a simple light switch analogy. Let's use a more nuanced analogy for the deep dive:

**Scenario: Bank Teller Processing Transactions**

Imagine a bank teller processing your request to transfer money.

**Simple Idempotency (Phase 1 level):**
- Teller writes your request on a numbered slip
- If you come back with the same slip number, they show you the receipt from before
- Problem: What if the teller was halfway through the transfer when the power went out?

**Advanced Idempotency (This topic):**
1. **Transaction Log**: Teller writes each step in a ledger
   - "10:00:00 - Request #123 - Started transfer A→B $100"
   - "10:00:01 - Request #123 - Debited A"
   - "10:00:02 - Request #123 - Credited B"
   - "10:00:03 - Request #123 - Completed"

2. **Recovery**: If power goes out after step 2:
   - On restart, teller checks ledger
   - Sees #123 is incomplete
   - Either completes it or rolls it back
   - Then handles the retry correctly

3. **Scoping**: Each teller has their own slip numbers
   - Teller A's slip #123 ≠ Teller B's slip #123
   - Prevents collisions between different contexts

### The Idempotency Spectrum

**Idempotency Spectrum**

**NATURALLY IDEMPOTENT**
- SET x = 5 (absolute value)
- DELETE WHERE id = 123
- PUT /users/123 {name: "John"}

**IDEMPOTENT WITH KEY**
- Create order with idempotency key
- Process payment with request ID
- Send notification with message ID

**IDEMPOTENT WITH VERSIONING**
- Update if version matches
- Conditional writes (ETags)
- Compare-and-swap operations

**IDEMPOTENT WITH SAGA**
- Multi-step workflows with compensation
- Distributed transactions
- Event sourcing with deduplication

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────┐
│                  IDEMPOTENCY SPECTRUM                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  NATURALLY IDEMPOTENT                                       │
│  └─ SET x = 5 (absolute value)                              │
│  └─ DELETE WHERE id = 123                                   │
│  └─ PUT /users/123 {name: "John"}                           │
│                                                              │
│  IDEMPOTENT WITH KEY                                        │
│  └─ Create order with idempotency key                       │
│  └─ Process payment with request ID                         │
│  └─ Send notification with message ID                       │
│                                                              │
│  IDEMPOTENT WITH VERSIONING                                 │
│  └─ Update if version matches                               │
│  └─ Conditional writes (ETags)                              │
│  └─ Compare-and-swap operations                             │
│                                                              │
│  IDEMPOTENT WITH SAGA                                       │
│  └─ Multi-step workflows with compensation                  │
│  └─ Distributed transactions                                │
│  └─ Event sourcing with deduplication                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```
</details>

---

## 3️⃣ How It Works Internally

### Pattern 1: Idempotency Key with Request Fingerprinting

The basic pattern from Phase 1 stores the response. The advanced pattern also validates the request:

```java
package com.systemdesign.idempotency;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.time.Duration;
import java.util.HexFormat;
import java.util.Optional;

/**
 * Advanced idempotency store with request fingerprinting.
 * Ensures the same idempotency key is always used with the same request body.
 */
@Component
public class AdvancedIdempotencyStore {
    
    private static final String KEY_PREFIX = "idempotency:";
    private static final Duration DEFAULT_TTL = Duration.ofHours(24);
    
    private final StringRedisTemplate redis;
    private final ObjectMapper objectMapper;
    
    public AdvancedIdempotencyStore(StringRedisTemplate redis, ObjectMapper objectMapper) {
        this.redis = redis;
        this.objectMapper = objectMapper;
    }
    
    /**
     * Try to acquire an idempotency lock.
     * Returns the stored result if this is a duplicate, or empty if this is new.
     */
    public AcquireResult tryAcquire(String apiKey, String idempotencyKey, 
                                     Object requestBody) {
        // Scope key by API key to prevent collisions between users
        String scopedKey = KEY_PREFIX + apiKey + ":" + idempotencyKey;
        
        // Calculate request fingerprint
        String fingerprint = calculateFingerprint(requestBody);
        
        // Try to get existing entry
        String existing = redis.opsForValue().get(scopedKey);
        
        if (existing != null) {
            IdempotencyEntry entry = deserialize(existing);
            
            // Check if request body matches
            if (!fingerprint.equals(entry.requestFingerprint)) {
                // Same key, different request = error!
                return AcquireResult.mismatch(
                    "Idempotency key already used with different request body");
            }
            
            // Check status
            return switch (entry.status) {
                case PROCESSING -> AcquireResult.inProgress();
                case COMPLETED -> AcquireResult.duplicate(entry.response);
                case FAILED -> AcquireResult.canRetry();
            };
        }
        
        // New request, try to acquire lock
        IdempotencyEntry newEntry = new IdempotencyEntry(
            Status.PROCESSING, fingerprint, null, null);
        
        Boolean acquired = redis.opsForValue()
            .setIfAbsent(scopedKey, serialize(newEntry), DEFAULT_TTL);
        
        if (Boolean.TRUE.equals(acquired)) {
            return AcquireResult.acquired();
        }
        
        // Race condition: another request acquired it
        return AcquireResult.inProgress();
    }
    
    /**
     * Store the successful result.
     */
    public void storeSuccess(String apiKey, String idempotencyKey, 
                             int httpStatus, Object responseBody) {
        String scopedKey = KEY_PREFIX + apiKey + ":" + idempotencyKey;
        
        // Get existing entry to preserve fingerprint
        String existing = redis.opsForValue().get(scopedKey);
        IdempotencyEntry entry = deserialize(existing);
        
        IdempotencyEntry updated = new IdempotencyEntry(
            Status.COMPLETED, 
            entry.requestFingerprint,
            httpStatus,
            responseBody
        );
        
        redis.opsForValue().set(scopedKey, serialize(updated), DEFAULT_TTL);
    }
    
    /**
     * Mark as failed to allow retry.
     */
    public void markFailed(String apiKey, String idempotencyKey) {
        String scopedKey = KEY_PREFIX + apiKey + ":" + idempotencyKey;
        
        String existing = redis.opsForValue().get(scopedKey);
        if (existing != null) {
            IdempotencyEntry entry = deserialize(existing);
            IdempotencyEntry updated = new IdempotencyEntry(
                Status.FAILED, entry.requestFingerprint, null, null);
            redis.opsForValue().set(scopedKey, serialize(updated), Duration.ofMinutes(5));
        }
    }
    
    /**
     * Calculate a fingerprint of the request body.
     * This ensures the same idempotency key is used with the same request.
     */
    private String calculateFingerprint(Object requestBody) {
        try {
            String json = objectMapper.writeValueAsString(requestBody);
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(json.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hash);
        } catch (Exception e) {
            throw new RuntimeException("Failed to calculate fingerprint", e);
        }
    }
    
    private String serialize(IdempotencyEntry entry) {
        try {
            return objectMapper.writeValueAsString(entry);
        } catch (Exception e) {
            throw new RuntimeException("Failed to serialize", e);
        }
    }
    
    private IdempotencyEntry deserialize(String json) {
        try {
            return objectMapper.readValue(json, IdempotencyEntry.class);
        } catch (Exception e) {
            throw new RuntimeException("Failed to deserialize", e);
        }
    }
    
    // Inner classes
    
    public enum Status { PROCESSING, COMPLETED, FAILED }
    
    public record IdempotencyEntry(
        Status status,
        String requestFingerprint,
        Integer httpStatus,
        Object response
    ) {}
    
    public sealed interface AcquireResult {
        record Acquired() implements AcquireResult {}
        record Duplicate(Object response) implements AcquireResult {}
        record InProgress() implements AcquireResult {}
        record Mismatch(String error) implements AcquireResult {}
        record CanRetry() implements AcquireResult {}
        
        static AcquireResult acquired() { return new Acquired(); }
        static AcquireResult duplicate(Object response) { return new Duplicate(response); }
        static AcquireResult inProgress() { return new InProgress(); }
        static AcquireResult mismatch(String error) { return new Mismatch(error); }
        static AcquireResult canRetry() { return new CanRetry(); }
    }
}
```

### Pattern 2: Database-Level Idempotency with Transactions

For critical operations, combine application-level idempotency with database constraints:

```java
package com.systemdesign.idempotency;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.Optional;

/**
 * Database-backed idempotency that survives application restarts.
 * Uses database transactions for atomicity.
 */
@Repository
public class DatabaseIdempotencyStore {
    
    private final JdbcTemplate jdbc;
    
    public DatabaseIdempotencyStore(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }
    
    /**
     * Execute an operation idempotently using database transactions.
     * The operation and idempotency record are in the same transaction.
     */
    @Transactional
    public <T> T executeIdempotent(String idempotencyKey, String operationType,
                                    IdempotentOperation<T> operation) {
        // Check if already processed
        Optional<IdempotencyRecord> existing = findByKey(idempotencyKey);
        
        if (existing.isPresent()) {
            IdempotencyRecord record = existing.get();
            
            if (record.status() == Status.COMPLETED) {
                // Already done, return stored result
                return deserializeResult(record.result());
            }
            
            if (record.status() == Status.PROCESSING) {
                // Check if it's stale (process crashed)
                if (isStale(record)) {
                    // Clean up and retry
                    deleteRecord(idempotencyKey);
                } else {
                    throw new ConcurrentRequestException(
                        "Request is already being processed");
                }
            }
            
            // Status is FAILED, allow retry
        }
        
        // Insert processing record (will fail if duplicate due to unique constraint)
        try {
            insertRecord(idempotencyKey, operationType, Status.PROCESSING);
        } catch (org.springframework.dao.DuplicateKeyException e) {
            // Another request beat us, retry the check
            return executeIdempotent(idempotencyKey, operationType, operation);
        }
        
        try {
            // Execute the actual operation
            T result = operation.execute();
            
            // Update record with result
            updateRecord(idempotencyKey, Status.COMPLETED, serializeResult(result));
            
            return result;
            
        } catch (Exception e) {
            // Mark as failed
            updateRecord(idempotencyKey, Status.FAILED, e.getMessage());
            throw e;
        }
    }
    
    private Optional<IdempotencyRecord> findByKey(String key) {
        var results = jdbc.query(
            "SELECT idempotency_key, operation_type, status, result, created_at " +
            "FROM idempotency_records WHERE idempotency_key = ?",
            (rs, rowNum) -> new IdempotencyRecord(
                rs.getString("idempotency_key"),
                rs.getString("operation_type"),
                Status.valueOf(rs.getString("status")),
                rs.getString("result"),
                rs.getTimestamp("created_at").toInstant()
            ),
            key
        );
        return results.isEmpty() ? Optional.empty() : Optional.of(results.get(0));
    }
    
    private void insertRecord(String key, String operationType, Status status) {
        jdbc.update(
            "INSERT INTO idempotency_records (idempotency_key, operation_type, status, created_at) " +
            "VALUES (?, ?, ?, NOW())",
            key, operationType, status.name()
        );
    }
    
    private void updateRecord(String key, Status status, String result) {
        jdbc.update(
            "UPDATE idempotency_records SET status = ?, result = ?, updated_at = NOW() " +
            "WHERE idempotency_key = ?",
            status.name(), result, key
        );
    }
    
    private void deleteRecord(String key) {
        jdbc.update("DELETE FROM idempotency_records WHERE idempotency_key = ?", key);
    }
    
    private boolean isStale(IdempotencyRecord record) {
        // Consider processing records older than 5 minutes as stale
        return record.createdAt().isBefore(Instant.now().minusSeconds(300));
    }
    
    @SuppressWarnings("unchecked")
    private <T> T deserializeResult(String result) {
        // In production, use proper serialization
        return (T) result;
    }
    
    private String serializeResult(Object result) {
        // In production, use proper serialization
        return result != null ? result.toString() : null;
    }
    
    public enum Status { PROCESSING, COMPLETED, FAILED }
    
    public record IdempotencyRecord(
        String idempotencyKey,
        String operationType,
        Status status,
        String result,
        Instant createdAt
    ) {}
    
    @FunctionalInterface
    public interface IdempotentOperation<T> {
        T execute() throws Exception;
    }
    
    public static class ConcurrentRequestException extends RuntimeException {
        public ConcurrentRequestException(String message) {
            super(message);
        }
    }
}
```

**SQL Schema:**

```sql
CREATE TABLE idempotency_records (
    idempotency_key VARCHAR(255) PRIMARY KEY,
    operation_type VARCHAR(100) NOT NULL,
    status VARCHAR(20) NOT NULL,
    result TEXT,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP,
    
    INDEX idx_status_created (status, created_at)
);
```

### Pattern 3: Idempotency for Multi-Step Operations (Saga Pattern)

For operations that span multiple services:

```java
package com.systemdesign.idempotency;

import java.util.*;
import java.util.concurrent.*;

/**
 * Saga pattern with idempotency for multi-step distributed operations.
 * Each step is idempotent, and the entire saga can be safely retried.
 */
public class IdempotentSaga<T> {
    
    private final String sagaId;
    private final SagaStore store;
    private final List<SagaStep<T>> steps;
    
    public IdempotentSaga(String sagaId, SagaStore store) {
        this.sagaId = sagaId;
        this.store = store;
        this.steps = new ArrayList<>();
    }
    
    /**
     * Add a step to the saga.
     * Each step has an action and a compensation (rollback).
     */
    public IdempotentSaga<T> addStep(String stepId, 
                                      StepAction<T> action, 
                                      StepCompensation<T> compensation) {
        steps.add(new SagaStep<>(stepId, action, compensation));
        return this;
    }
    
    /**
     * Execute the saga idempotently.
     * If the saga was already executed, returns the stored result.
     * If the saga was partially executed, resumes from where it left off.
     */
    public SagaResult<T> execute(T context) {
        // Load or create saga state
        SagaState state = store.loadOrCreate(sagaId);
        
        if (state.status() == SagaStatus.COMPLETED) {
            // Already completed successfully
            return SagaResult.success(state.result());
        }
        
        if (state.status() == SagaStatus.COMPENSATED) {
            // Already rolled back
            return SagaResult.rolledBack(state.error());
        }
        
        // Execute or resume
        int startStep = state.completedSteps().size();
        List<String> completedSteps = new ArrayList<>(state.completedSteps());
        
        try {
            for (int i = startStep; i < steps.size(); i++) {
                SagaStep<T> step = steps.get(i);
                
                // Check if step was already completed (idempotency)
                if (completedSteps.contains(step.stepId())) {
                    continue;
                }
                
                // Execute step
                step.action().execute(context);
                
                // Mark step as completed
                completedSteps.add(step.stepId());
                store.updateProgress(sagaId, completedSteps, SagaStatus.IN_PROGRESS);
            }
            
            // All steps completed
            store.complete(sagaId, context);
            return SagaResult.success(context);
            
        } catch (Exception e) {
            // Step failed, compensate
            return compensate(context, completedSteps, e);
        }
    }
    
    /**
     * Compensate (rollback) completed steps in reverse order.
     */
    private SagaResult<T> compensate(T context, List<String> completedSteps, Exception error) {
        store.updateProgress(sagaId, completedSteps, SagaStatus.COMPENSATING);
        
        // Compensate in reverse order
        List<String> stepsToCompensate = new ArrayList<>(completedSteps);
        Collections.reverse(stepsToCompensate);
        
        for (String stepId : stepsToCompensate) {
            SagaStep<T> step = findStep(stepId);
            if (step != null && step.compensation() != null) {
                try {
                    step.compensation().compensate(context);
                } catch (Exception compensationError) {
                    // Log but continue compensating other steps
                    System.err.println("Compensation failed for step " + stepId + 
                        ": " + compensationError.getMessage());
                }
            }
        }
        
        store.markCompensated(sagaId, error.getMessage());
        return SagaResult.rolledBack(error.getMessage());
    }
    
    private SagaStep<T> findStep(String stepId) {
        return steps.stream()
            .filter(s -> s.stepId().equals(stepId))
            .findFirst()
            .orElse(null);
    }
    
    // Inner types
    
    public record SagaStep<T>(
        String stepId,
        StepAction<T> action,
        StepCompensation<T> compensation
    ) {}
    
    @FunctionalInterface
    public interface StepAction<T> {
        void execute(T context) throws Exception;
    }
    
    @FunctionalInterface
    public interface StepCompensation<T> {
        void compensate(T context) throws Exception;
    }
    
    public enum SagaStatus {
        IN_PROGRESS, COMPLETED, COMPENSATING, COMPENSATED
    }
    
    public record SagaState(
        String sagaId,
        SagaStatus status,
        List<String> completedSteps,
        Object result,
        String error
    ) {}
    
    public sealed interface SagaResult<T> {
        record Success<T>(T result) implements SagaResult<T> {}
        record RolledBack<T>(String error) implements SagaResult<T> {}
        
        static <T> SagaResult<T> success(T result) { return new Success<>(result); }
        static <T> SagaResult<T> rolledBack(String error) { return new RolledBack<>(error); }
    }
    
    public interface SagaStore {
        SagaState loadOrCreate(String sagaId);
        void updateProgress(String sagaId, List<String> completedSteps, SagaStatus status);
        void complete(String sagaId, Object result);
        void markCompensated(String sagaId, String error);
    }
}
```

**Usage Example:**

```java
package com.systemdesign.idempotency;

/**
 * Example: Order processing saga with idempotency.
 */
public class OrderProcessingSaga {
    
    private final SagaStore sagaStore;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;
    
    public OrderProcessingSaga(SagaStore sagaStore,
                                InventoryService inventoryService,
                                PaymentService paymentService,
                                ShippingService shippingService) {
        this.sagaStore = sagaStore;
        this.inventoryService = inventoryService;
        this.paymentService = paymentService;
        this.shippingService = shippingService;
    }
    
    public IdempotentSaga.SagaResult<OrderContext> processOrder(String orderId, 
                                                                 OrderRequest request) {
        OrderContext context = new OrderContext(orderId, request);
        
        IdempotentSaga<OrderContext> saga = new IdempotentSaga<>(
            "order-" + orderId, sagaStore)
            
            // Step 1: Reserve inventory
            .addStep("reserve-inventory",
                ctx -> {
                    String reservationId = inventoryService.reserve(
                        ctx.request().productId(),
                        ctx.request().quantity()
                    );
                    ctx.setReservationId(reservationId);
                },
                ctx -> inventoryService.cancelReservation(ctx.getReservationId())
            )
            
            // Step 2: Process payment
            .addStep("process-payment",
                ctx -> {
                    String paymentId = paymentService.charge(
                        ctx.request().customerId(),
                        ctx.request().amount(),
                        "order-" + ctx.orderId() // Idempotency key for payment
                    );
                    ctx.setPaymentId(paymentId);
                },
                ctx -> paymentService.refund(ctx.getPaymentId())
            )
            
            // Step 3: Create shipment
            .addStep("create-shipment",
                ctx -> {
                    String shipmentId = shippingService.createShipment(
                        ctx.orderId(),
                        ctx.request().shippingAddress()
                    );
                    ctx.setShipmentId(shipmentId);
                },
                ctx -> shippingService.cancelShipment(ctx.getShipmentId())
            );
        
        return saga.execute(context);
    }
    
    // Context and supporting classes
    
    public static class OrderContext {
        private final String orderId;
        private final OrderRequest request;
        private String reservationId;
        private String paymentId;
        private String shipmentId;
        
        public OrderContext(String orderId, OrderRequest request) {
            this.orderId = orderId;
            this.request = request;
        }
        
        public String orderId() { return orderId; }
        public OrderRequest request() { return request; }
        
        public String getReservationId() { return reservationId; }
        public void setReservationId(String id) { this.reservationId = id; }
        
        public String getPaymentId() { return paymentId; }
        public void setPaymentId(String id) { this.paymentId = id; }
        
        public String getShipmentId() { return shipmentId; }
        public void setShipmentId(String id) { this.shipmentId = id; }
    }
    
    public record OrderRequest(
        String productId,
        int quantity,
        String customerId,
        double amount,
        String shippingAddress
    ) {}
    
    // Service interfaces
    public interface InventoryService {
        String reserve(String productId, int quantity);
        void cancelReservation(String reservationId);
    }
    
    public interface PaymentService {
        String charge(String customerId, double amount, String idempotencyKey);
        void refund(String paymentId);
    }
    
    public interface ShippingService {
        String createShipment(String orderId, String address);
        void cancelShipment(String shipmentId);
    }
    
    public interface SagaStore extends IdempotentSaga.SagaStore {}
}
```

### Pattern 4: Event Sourcing with Deduplication

For event-driven systems:

```java
package com.systemdesign.idempotency;

import java.time.Instant;
import java.util.*;
import java.util.concurrent.*;

/**
 * Event store with built-in deduplication.
 * Events are identified by their event ID, preventing duplicates.
 */
public class IdempotentEventStore {
    
    private final ConcurrentMap<String, List<Event>> streamStore = new ConcurrentHashMap<>();
    private final Set<String> processedEventIds = ConcurrentHashMap.newKeySet();
    
    /**
     * Append an event to a stream, with idempotency.
     * If the event ID was already processed, this is a no-op.
     */
    public AppendResult append(String streamId, Event event) {
        // Check if event was already processed
        if (processedEventIds.contains(event.eventId())) {
            return new AppendResult(false, "Event already processed");
        }
        
        // Try to mark as processed (atomic)
        if (!processedEventIds.add(event.eventId())) {
            // Another thread beat us
            return new AppendResult(false, "Event already processed");
        }
        
        // Append to stream
        streamStore.computeIfAbsent(streamId, k -> new CopyOnWriteArrayList<>())
            .add(event);
        
        return new AppendResult(true, null);
    }
    
    /**
     * Read events from a stream.
     */
    public List<Event> readStream(String streamId) {
        return streamStore.getOrDefault(streamId, Collections.emptyList());
    }
    
    /**
     * Read events from a stream starting from a specific position.
     */
    public List<Event> readStream(String streamId, int fromPosition) {
        List<Event> events = streamStore.getOrDefault(streamId, Collections.emptyList());
        if (fromPosition >= events.size()) {
            return Collections.emptyList();
        }
        return events.subList(fromPosition, events.size());
    }
    
    /**
     * Process events idempotently.
     * Each event is processed at most once.
     */
    public void processEvents(String streamId, EventProcessor processor) {
        List<Event> events = readStream(streamId);
        
        for (Event event : events) {
            String processingKey = "processed:" + processor.processorId() + ":" + event.eventId();
            
            if (processedEventIds.contains(processingKey)) {
                continue; // Already processed by this processor
            }
            
            if (processedEventIds.add(processingKey)) {
                try {
                    processor.process(event);
                } catch (Exception e) {
                    // Remove from processed set to allow retry
                    processedEventIds.remove(processingKey);
                    throw e;
                }
            }
        }
    }
    
    public record Event(
        String eventId,
        String eventType,
        Object data,
        Instant timestamp,
        Map<String, String> metadata
    ) {
        public static Event create(String eventType, Object data) {
            return new Event(
                UUID.randomUUID().toString(),
                eventType,
                data,
                Instant.now(),
                new HashMap<>()
            );
        }
    }
    
    public record AppendResult(boolean success, String error) {}
    
    @FunctionalInterface
    public interface EventProcessor {
        String processorId();
        void process(Event event);
    }
}
```

---

## 4️⃣ Simulation-First Explanation

Let's trace through a complex idempotency scenario.

### Scenario: Order Processing with Partial Failure

**Setup:**
- Order service processes orders
- Each order involves: inventory reservation, payment, shipping
- Network is unreliable

### First Attempt (Partial Failure)

```
Time 0ms: Client sends "Create Order #123"
          Idempotency-Key: "order-123-create"

Time 5ms: Server receives request
  - Checks idempotency store: not found
  - Acquires lock: idempotency:order-123-create = PROCESSING
  
Time 10ms: Step 1 - Reserve Inventory
  - Calls inventory service
  - Reservation successful: RES-456
  - Updates saga state: ["reserve-inventory"]

Time 20ms: Step 2 - Process Payment
  - Calls payment service with idempotency key "order-123-payment"
  - Payment successful: PAY-789
  - Updates saga state: ["reserve-inventory", "process-payment"]

Time 30ms: Step 3 - Create Shipment
  - Calls shipping service
  - SERVER CRASHES!
  
Time 35ms: Client timeout, no response received
```

### Client Retry

```
Time 5000ms: Client retries with same idempotency key
            Idempotency-Key: "order-123-create"

Time 5005ms: Server (restarted) receives request
  - Checks idempotency store: found!
  - Status: PROCESSING
  - Created: 5000ms ago (stale!)
  - Cleans up stale record
  
Time 5010ms: Loads saga state from database
  - Saga ID: "order-123"
  - Status: IN_PROGRESS
  - Completed steps: ["reserve-inventory", "process-payment"]
  
Time 5015ms: Resumes from Step 3
  - Step 1 (reserve-inventory): Already done, skip
  - Step 2 (process-payment): Already done, skip
  - Step 3 (create-shipment): Execute
  
Time 5020ms: Step 3 - Create Shipment
  - Calls shipping service
  - Shipment created: SHIP-012
  - Updates saga state: ["reserve-inventory", "process-payment", "create-shipment"]

Time 5025ms: Saga completed
  - Updates idempotency store: COMPLETED
  - Returns success to client

Client receives: Order #123 created successfully
```

### What If Step 2 Had Failed?

```
Alternative timeline:

Time 20ms: Step 2 - Process Payment
  - Calls payment service
  - Payment FAILS (insufficient funds)
  
Time 25ms: Saga compensation begins
  - Step 1 compensation: Cancel inventory reservation
  - Calls inventory service: cancelReservation(RES-456)
  
Time 30ms: Saga compensated
  - Updates saga state: COMPENSATED
  - Updates idempotency store: COMPLETED (with error)
  - Returns error to client

Time 5000ms: Client retries
  - Idempotency store shows: COMPLETED
  - Returns cached error response
  
Client receives: Order failed (insufficient funds)
No duplicate inventory reservation!
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Stripe's Idempotency Implementation

Stripe's approach is considered the gold standard:

```
1. Idempotency key in header
   Idempotency-Key: <unique-key>

2. Scoped by API key
   Each merchant's keys are isolated

3. Request body fingerprinting
   Same key + different body = 400 error

4. 24-hour TTL
   Keys expire after 24 hours

5. In-flight detection
   If request is processing, return 409 Conflict

6. Detailed responses
   X-Idempotency-Replayed: true (for cached responses)
```

### Uber's Payment Idempotency

Uber's approach for high-volume payments:

```
1. Multiple layers of idempotency
   - API Gateway: Deduplicates at edge
   - Service layer: Application-level idempotency
   - Database: Unique constraints

2. Idempotency key structure
   trip_id + action + timestamp_bucket
   Example: "trip-123-charge-2024010112"

3. Reconciliation
   - Periodic jobs check for inconsistencies
   - Alerts on duplicate charges
   - Automatic refunds for duplicates
```

### AWS Lambda Idempotency

AWS provides a library for Lambda idempotency:

```python
# AWS Lambda Powertools for Python
from aws_lambda_powertools.utilities.idempotency import (
    idempotent, DynamoDBPersistenceLayer
)

persistence_layer = DynamoDBPersistenceLayer(table_name="IdempotencyTable")

@idempotent(persistence_store=persistence_layer)
def handler(event, context):
    # This function is now idempotent
    # Duplicate invocations return cached result
    return process_payment(event)
```

### Production Checklist

```
┌─────────────────────────────────────────────────────────────┐
│          IDEMPOTENCY PRODUCTION CHECKLIST                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  KEY GENERATION                                             │
│  □ Client generates key before first attempt                │
│  □ Key is reused for all retries                            │
│  □ Key includes enough context (user, operation, timestamp) │
│  □ Key is scoped (by user/API key)                          │
│                                                              │
│  STORAGE                                                    │
│  □ Atomic check-and-set (prevent races)                     │
│  □ Reasonable TTL (24-48 hours typical)                     │
│  □ Survives restarts (Redis with persistence or DB)         │
│  □ Handles stale "processing" entries                       │
│                                                              │
│  REQUEST VALIDATION                                         │
│  □ Fingerprint request body                                 │
│  □ Reject same key with different body                      │
│  □ Return clear error messages                              │
│                                                              │
│  RESPONSE HANDLING                                          │
│  □ Store complete response (status + body)                  │
│  □ Indicate replayed responses (header)                     │
│  □ Handle partial failures                                  │
│                                                              │
│  MONITORING                                                 │
│  □ Track duplicate request rate                             │
│  □ Alert on high duplicate rate (might indicate bug)        │
│  □ Monitor storage usage                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6️⃣ How to Implement or Apply It

### Complete Spring Boot Implementation

#### Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
</dependencies>
```

#### Idempotency Annotation

```java
package com.systemdesign.idempotency;

import java.lang.annotation.*;

/**
 * Marks a method as idempotent.
 * The method will be executed at most once for each unique idempotency key.
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Idempotent {
    
    /**
     * SpEL expression to extract the idempotency key from method arguments.
     * Example: "#request.orderId" or "#headers['Idempotency-Key']"
     */
    String key();
    
    /**
     * TTL for the idempotency record in seconds.
     */
    long ttlSeconds() default 86400; // 24 hours
    
    /**
     * Whether to validate that the request body matches.
     */
    boolean validateBody() default true;
}
```

#### Idempotency Aspect

```java
package com.systemdesign.idempotency;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.core.DefaultParameterNameDiscoverer;
import org.springframework.expression.EvaluationContext;
import org.springframework.expression.ExpressionParser;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

/**
 * Aspect that implements idempotency for annotated methods.
 */
@Aspect
@Component
public class IdempotencyAspect {
    
    private final AdvancedIdempotencyStore idempotencyStore;
    private final ObjectMapper objectMapper;
    private final ExpressionParser parser = new SpelExpressionParser();
    private final DefaultParameterNameDiscoverer parameterNameDiscoverer = 
        new DefaultParameterNameDiscoverer();
    
    public IdempotencyAspect(AdvancedIdempotencyStore idempotencyStore, 
                             ObjectMapper objectMapper) {
        this.idempotencyStore = idempotencyStore;
        this.objectMapper = objectMapper;
    }
    
    @Around("@annotation(idempotent)")
    public Object handleIdempotency(ProceedingJoinPoint joinPoint, 
                                     Idempotent idempotent) throws Throwable {
        
        // Extract idempotency key using SpEL
        String idempotencyKey = extractKey(joinPoint, idempotent.key());
        
        if (idempotencyKey == null || idempotencyKey.isBlank()) {
            // No idempotency key, proceed normally
            return joinPoint.proceed();
        }
        
        // Get API key from context (simplified, would come from auth)
        String apiKey = getApiKey();
        
        // Get request body for fingerprinting
        Object requestBody = getRequestBody(joinPoint);
        
        // Try to acquire idempotency lock
        var result = idempotencyStore.tryAcquire(apiKey, idempotencyKey, requestBody);
        
        return switch (result) {
            case AdvancedIdempotencyStore.AcquireResult.Acquired() -> {
                try {
                    Object response = joinPoint.proceed();
                    
                    // Store successful result
                    int status = extractHttpStatus(response);
                    Object body = extractResponseBody(response);
                    idempotencyStore.storeSuccess(apiKey, idempotencyKey, status, body);
                    
                    yield response;
                } catch (Exception e) {
                    idempotencyStore.markFailed(apiKey, idempotencyKey);
                    throw e;
                }
            }
            
            case AdvancedIdempotencyStore.AcquireResult.Duplicate(Object cachedResponse) -> {
                // Return cached response
                yield ResponseEntity.ok()
                    .header("X-Idempotency-Replayed", "true")
                    .body(cachedResponse);
            }
            
            case AdvancedIdempotencyStore.AcquireResult.InProgress() -> {
                yield ResponseEntity.status(409)
                    .body(new ErrorResponse("Request is already being processed"));
            }
            
            case AdvancedIdempotencyStore.AcquireResult.Mismatch(String error) -> {
                yield ResponseEntity.status(422)
                    .body(new ErrorResponse(error));
            }
            
            case AdvancedIdempotencyStore.AcquireResult.CanRetry() -> {
                // Previous attempt failed, retry
                yield joinPoint.proceed();
            }
        };
    }
    
    private String extractKey(ProceedingJoinPoint joinPoint, String keyExpression) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        
        EvaluationContext context = new StandardEvaluationContext();
        String[] parameterNames = parameterNameDiscoverer.getParameterNames(method);
        Object[] args = joinPoint.getArgs();
        
        if (parameterNames != null) {
            for (int i = 0; i < parameterNames.length; i++) {
                context.setVariable(parameterNames[i], args[i]);
            }
        }
        
        return parser.parseExpression(keyExpression).getValue(context, String.class);
    }
    
    private String getApiKey() {
        // In production, extract from security context
        return "default-api-key";
    }
    
    private Object getRequestBody(ProceedingJoinPoint joinPoint) {
        // Find the request body argument (typically annotated with @RequestBody)
        for (Object arg : joinPoint.getArgs()) {
            if (arg != null && !isPrimitiveOrWrapper(arg.getClass())) {
                return arg;
            }
        }
        return null;
    }
    
    private boolean isPrimitiveOrWrapper(Class<?> type) {
        return type.isPrimitive() || 
               type == String.class ||
               Number.class.isAssignableFrom(type);
    }
    
    private int extractHttpStatus(Object response) {
        if (response instanceof ResponseEntity<?> re) {
            return re.getStatusCode().value();
        }
        return 200;
    }
    
    private Object extractResponseBody(Object response) {
        if (response instanceof ResponseEntity<?> re) {
            return re.getBody();
        }
        return response;
    }
    
    private record ErrorResponse(String error) {}
}
```

#### Controller with Idempotency

```java
package com.systemdesign.idempotency;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    private final OrderService orderService;
    
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }
    
    /**
     * Create an order with idempotency.
     * Uses the Idempotency-Key header for deduplication.
     */
    @PostMapping
    @Idempotent(key = "#idempotencyKey", ttlSeconds = 86400)
    public ResponseEntity<OrderResponse> createOrder(
            @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey,
            @RequestBody CreateOrderRequest request) {
        
        OrderResponse response = orderService.createOrder(request);
        return ResponseEntity.ok(response);
    }
    
    /**
     * Process payment with idempotency.
     * Uses order ID + action as the idempotency key.
     */
    @PostMapping("/{orderId}/pay")
    @Idempotent(key = "'order-' + #orderId + '-pay'")
    public ResponseEntity<PaymentResponse> processPayment(
            @PathVariable String orderId,
            @RequestBody PaymentRequest request) {
        
        PaymentResponse response = orderService.processPayment(orderId, request);
        return ResponseEntity.ok(response);
    }
    
    // DTOs
    public record CreateOrderRequest(String productId, int quantity, String customerId) {}
    public record OrderResponse(String orderId, String status, double total) {}
    public record PaymentRequest(String paymentMethod, double amount) {}
    public record PaymentResponse(String paymentId, String status) {}
    
    // Service interface
    public interface OrderService {
        OrderResponse createOrder(CreateOrderRequest request);
        PaymentResponse processPayment(String orderId, PaymentRequest request);
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

#### 1. Idempotency Key Scope Too Narrow

**Wrong:**
```java
// BAD: Key only includes order ID
String key = "order-" + orderId;
// Two different operations on same order collide!
```

**Right:**
```java
// GOOD: Key includes operation type
String key = "order-" + orderId + "-" + operationType;
// "order-123-create" vs "order-123-cancel" are different
```

#### 2. Not Handling Partial Failures

**Wrong:**
```java
// BAD: No saga pattern
public void processOrder(Order order) {
    inventoryService.reserve(order);  // Succeeds
    paymentService.charge(order);     // Fails!
    // Inventory is reserved but payment failed
    // Retry will reserve again!
}
```

**Right:**
```java
// GOOD: Saga with compensation
public void processOrder(Order order) {
    saga.addStep("reserve", 
        () -> inventoryService.reserve(order),
        () -> inventoryService.cancelReservation(order));
    saga.addStep("charge",
        () -> paymentService.charge(order),
        () -> paymentService.refund(order));
    saga.execute();
}
```

#### 3. Idempotency Key in Request Body

**Wrong:**
```java
// BAD: Key in body might not be parsed on error
@PostMapping("/orders")
public Order createOrder(@RequestBody OrderRequest request) {
    String key = request.getIdempotencyKey(); // Body might fail to parse!
}
```

**Right:**
```java
// GOOD: Key in header
@PostMapping("/orders")
public Order createOrder(
        @RequestHeader("Idempotency-Key") String key,
        @RequestBody OrderRequest request) {
    // Key is extracted before body parsing
}
```

#### 4. Not Validating Request Body

**Wrong:**
```java
// BAD: Same key, different body accepted
if (store.exists(key)) {
    return store.getResult(key);
}
// User might send different request with same key!
```

**Right:**
```java
// GOOD: Validate body matches
if (store.exists(key)) {
    if (!store.bodyMatches(key, requestBody)) {
        throw new IdempotencyKeyReusedException();
    }
    return store.getResult(key);
}
```

### Performance Considerations

```
┌─────────────────────────────────────────────────────────────┐
│             IDEMPOTENCY PERFORMANCE IMPACT                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Per-request overhead:                                      │
│  - Redis check: ~1ms                                        │
│  - Fingerprint calculation: ~0.1ms                          │
│  - Redis store: ~1ms                                        │
│  - Total: ~2-3ms per request                                │
│                                                              │
│  Storage requirements:                                      │
│  - Key: ~100 bytes                                          │
│  - Fingerprint: 64 bytes (SHA-256)                          │
│  - Response: varies (compress if large)                     │
│  - Total per entry: ~500 bytes to several KB                │
│                                                              │
│  At scale (10K requests/second, 24h TTL):                   │
│  - Entries: 864M                                            │
│  - Storage: ~400GB to several TB                            │
│                                                              │
│  Optimizations:                                             │
│  - Shorter TTL for non-critical operations                  │
│  - Compress stored responses                                │
│  - Store only essential response fields                     │
│  - Use Redis Cluster for horizontal scaling                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8️⃣ When NOT to Use This

### When Idempotency is Unnecessary

1. **GET requests**: Already idempotent by definition
2. **Naturally idempotent operations**: SET x = 5, DELETE WHERE id = 123
3. **Idempotent by business logic**: "Mark order as shipped" (already shipped = no-op)

### When Simpler Alternatives Work

1. **Unique constraints**: Database prevents duplicates
2. **Optimistic locking**: Version numbers prevent conflicts
3. **Event deduplication**: Message broker handles duplicates

### Anti-Patterns

1. **Idempotency for everything**
   - Adds latency and complexity
   - Only use for non-idempotent operations

2. **User-controlled keys without validation**
   - Users might reuse keys incorrectly
   - Always scope by user/API key

3. **Infinite TTL**
   - Storage grows unbounded
   - Old keys become irrelevant

---

## 9️⃣ Comparison with Alternatives

### Idempotency Approaches Comparison

| Approach | Complexity | Durability | Use Case |
|----------|------------|------------|----------|
| Redis-based | Low | Medium | Most APIs |
| Database-based | Medium | High | Critical operations |
| Saga pattern | High | High | Multi-service operations |
| Event sourcing | High | High | Event-driven systems |

### When to Use Each

| Scenario | Recommended Approach |
|----------|---------------------|
| Simple API endpoint | Redis-based idempotency |
| Payment processing | Database + Redis (hybrid) |
| Order processing | Saga pattern |
| Event consumers | Event sourcing with deduplication |
| Batch processing | Checkpoint-based |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q1: What's the difference between idempotency keys and unique constraints?**

**Answer:**
Both prevent duplicates, but they work at different levels:

**Idempotency keys:**
- Application-level
- Prevent duplicate processing
- Store the response for replay
- Client provides the key
- Temporary (TTL-based)

**Unique constraints:**
- Database-level
- Prevent duplicate data
- Reject duplicates with error
- Based on data values
- Permanent

Use both together: idempotency key prevents duplicate processing, unique constraint is a safety net if idempotency fails.

**Q2: Why should the client generate the idempotency key, not the server?**

**Answer:**
The client must generate the key because:

1. **Retry scenario**: If the server generates the key in the response, and the response is lost, the client can't retry with the same key.

2. **Client knows intent**: The client knows if two requests are the "same" operation. The server can't distinguish between intentional duplicate and retry.

3. **Deterministic retries**: Client generates key once, uses it for all retries. Server would generate different keys each time.

Example:
```
Client generates: key = "order-123-create"
First request: Server processes, stores result with key
Response lost!
Retry: Client uses same key, server returns cached result
```

### L5 (Senior) Questions

**Q3: How would you handle idempotency for a multi-step operation that spans multiple services?**

**Answer:**
I would use the Saga pattern with idempotent steps:

1. **Saga orchestrator** tracks overall progress
2. Each step has its own idempotency key: `saga-{sagaId}-step-{stepId}`
3. Each step is individually idempotent
4. Compensating actions are also idempotent

Example for order processing:
```
Saga: order-123
Steps:
  1. reserve-inventory (idempotent: check reservation exists)
  2. charge-payment (idempotent: use payment idempotency key)
  3. create-shipment (idempotent: check shipment exists)
```

If the saga is retried:
- Completed steps are skipped (idempotency check)
- Failed step is retried
- If step fails permanently, compensate completed steps

**Q4: How do you handle the case where the operation succeeds but storing the idempotency result fails?**

**Answer:**
This is a critical edge case. Options:

1. **Same transaction** (if possible):
   - Store idempotency result in the same database transaction as the operation
   - Both succeed or both fail
   - Works for database operations, not for external services

2. **Two-phase approach**:
   - First: Store "processing" status
   - Then: Execute operation
   - Finally: Update to "completed"
   - On retry: Check actual state (did operation happen?)

3. **Reconciliation**:
   - Accept that duplicates might occur
   - Run periodic reconciliation jobs
   - Detect and fix duplicates after the fact
   - Good for non-critical operations

For critical operations (payments), I'd use approach 2 with additional checks:
```java
if (idempotencyStore.getStatus(key) == PROCESSING) {
    // Check if operation actually completed
    if (paymentService.exists(orderId)) {
        // Operation succeeded, update idempotency store
        idempotencyStore.storeSuccess(key, existingPayment);
        return existingPayment;
    }
    // Operation didn't complete, allow retry
}
```

### L6 (Staff) Questions

**Q5: Design an idempotency system for a globally distributed payment platform.**

**Answer:**
For global distribution with low latency:

**Architecture:**
```
Region A (US-East)          Region B (Europe)
┌─────────────────┐        ┌─────────────────┐
│ API Gateway     │        │ API Gateway     │
│ (idempotency    │        │ (idempotency    │
│  check)         │        │  check)         │
└────────┬────────┘        └────────┬────────┘
         │                          │
         ▼                          ▼
┌─────────────────┐        ┌─────────────────┐
│ Regional Redis  │◄──────►│ Regional Redis  │
│ (fast check)    │  sync  │ (fast check)    │
└────────┬────────┘        └────────┬────────┘
         │                          │
         ▼                          ▼
┌─────────────────────────────────────────────┐
│         Global Idempotency Database          │
│         (source of truth, async sync)        │
└─────────────────────────────────────────────┘
```

**Approach:**

1. **Regional Redis** for fast idempotency checks
   - Low latency (< 5ms)
   - Handles 99% of cases

2. **Global database** for durability
   - Async replication from regional Redis
   - Source of truth for conflicts

3. **Idempotency key routing**
   - Hash key to determine "home" region
   - Cross-region requests check home region

4. **Conflict resolution**
   - If two regions process same key simultaneously
   - First-write-wins based on timestamp
   - Reconciliation job detects and alerts

**Tradeoffs:**
- Regional latency is low
- Cross-region might have brief window for duplicates
- Reconciliation handles edge cases

**Q6: How would you test an idempotency implementation?**

**Answer:**
Multi-level testing approach:

**Unit tests:**
```java
@Test
void shouldReturnCachedResponseOnDuplicate() {
    // First request
    store.tryAcquire("key", body);
    store.storeSuccess("key", 200, response);
    
    // Duplicate request
    var result = store.tryAcquire("key", body);
    assertThat(result).isInstanceOf(Duplicate.class);
}

@Test
void shouldRejectDifferentBodyWithSameKey() {
    store.tryAcquire("key", body1);
    var result = store.tryAcquire("key", body2);
    assertThat(result).isInstanceOf(Mismatch.class);
}
```

**Integration tests:**
```java
@Test
void shouldHandleConcurrentRequests() {
    ExecutorService executor = Executors.newFixedThreadPool(10);
    String key = UUID.randomUUID().toString();
    AtomicInteger processedCount = new AtomicInteger();
    
    // Submit 10 concurrent requests with same key
    List<Future<?>> futures = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
        futures.add(executor.submit(() -> {
            if (service.process(key, request)) {
                processedCount.incrementAndGet();
            }
        }));
    }
    
    // Wait for all to complete
    futures.forEach(f -> f.get());
    
    // Only one should have been processed
    assertThat(processedCount.get()).isEqualTo(1);
}
```

**Chaos tests:**
- Kill service mid-processing
- Network partition between service and Redis
- Redis failover during request
- Clock skew between servers

**Production monitoring:**
- Duplicate request rate
- Idempotency key collision rate
- Storage usage growth
- Latency impact

---

## 1️⃣1️⃣ One Clean Mental Summary

Idempotency in distributed systems goes beyond simple duplicate detection. Advanced patterns include request fingerprinting (ensuring the same key is used with the same request), database-level idempotency (for durability), saga patterns (for multi-step operations), and event sourcing with deduplication (for event-driven systems). The key challenges are partial failures (operation succeeds but response is lost), cross-service idempotency (maintaining consistency across services), and key scoping (preventing collisions between users). Production implementations like Stripe's combine multiple layers: API key scoping, request fingerprinting, in-flight detection, and detailed error responses. The critical insight is that idempotency is not just about storing responses; it's about ensuring that the entire operation, including all its side effects, happens exactly once regardless of how many times the request is sent.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│           ADVANCED IDEMPOTENCY CHEAT SHEET                   │
├─────────────────────────────────────────────────────────────┤
│ PATTERNS                                                     │
│   1. Key + Fingerprint: Validate request body matches       │
│   2. Database + Redis: Durability + speed                   │
│   3. Saga: Multi-step with compensation                     │
│   4. Event sourcing: Deduplication by event ID              │
├─────────────────────────────────────────────────────────────┤
│ KEY GENERATION                                               │
│   Format: {scope}-{entity}-{operation}-{timestamp}          │
│   Example: "user-123-order-456-create-20240101"             │
│   Always include: user/API key scope                        │
├─────────────────────────────────────────────────────────────┤
│ STATUSES                                                     │
│   PROCESSING: Request in flight, return 409                 │
│   COMPLETED: Return cached response                         │
│   FAILED: Allow retry                                       │
├─────────────────────────────────────────────────────────────┤
│ PARTIAL FAILURE HANDLING                                     │
│   1. Store progress in saga state                           │
│   2. On retry, resume from last completed step              │
│   3. Compensate completed steps on failure                  │
│   4. Each step is individually idempotent                   │
├─────────────────────────────────────────────────────────────┤
│ PRODUCTION CHECKLIST                                         │
│   □ Scope keys by user/API key                              │
│   □ Fingerprint request body                                │
│   □ Handle stale "processing" entries                       │
│   □ Set reasonable TTL (24-48 hours)                        │
│   □ Indicate replayed responses (header)                    │
│   □ Monitor duplicate rate                                  │
├─────────────────────────────────────────────────────────────┤
│ COMMON MISTAKES                                              │
│   ✗ Key in request body (might not parse)                   │
│   ✗ No request validation (different body, same key)        │
│   ✗ No partial failure handling                             │
│   ✗ Infinite TTL (storage grows forever)                    │
│   ✗ User-controlled keys without scoping                    │
└─────────────────────────────────────────────────────────────┘
```

