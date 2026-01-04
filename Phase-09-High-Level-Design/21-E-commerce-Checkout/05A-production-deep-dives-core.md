# E-commerce Checkout - Production Deep Dives (Core)

## Overview

This document covers the core production components: asynchronous messaging (Kafka for order events), caching strategy (Redis for carts and inventory), and the Saga pattern for distributed transactions.

---

## 1. Async, Messaging & Event Flow (Kafka)

### A) CONCEPT: What is Kafka in E-commerce Checkout Context?

In an e-commerce checkout system, Kafka serves as the event backbone for:

1. **Order Events**: Stream order lifecycle events (created, confirmed, shipped, delivered)
2. **Payment Events**: Stream payment processing events
3. **Inventory Events**: Stream inventory updates
4. **Notification Events**: Trigger email/SMS notifications

### B) OUR USAGE: How We Use Kafka Here

**Topic Design:**

| Topic | Purpose | Partitions | Key |
|-------|---------|------------|-----|
| `order-events` | Order lifecycle events | 10 | order_id |
| `payment-events` | Payment processing events | 5 | payment_id |
| `inventory-events` | Inventory updates | 5 | product_id |
| `notification-events` | Notification triggers | 10 | user_id |

**Why Order ID as Partition Key?**

```java
public class OrderEventPartitioner {
    
    public int partition(String orderId, int numPartitions) {
        return Math.abs(orderId.hashCode()) % numPartitions;
    }
}
```

Benefits:
- All events for same order go to same partition
- Maintains event ordering per order
- Simplifies event replay for specific orders

**Producer Configuration:**

```java
@Configuration
public class OrderKafkaConfig {
    
    @Bean
    public ProducerFactory<String, OrderEvent> orderEventProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // Durability settings (orders are critical)
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Kafka Technology-Level

**Normal Flow: Order Creation Event**

```
Step 1: Order Created
┌─────────────────────────────────────────────────────────────┐
│ Order Service: Creates order 'order_123456'                │
│ - Status: 'confirmed'                                       │
│ - Total: $221.97                                            │
│ - User: user_789                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Publish Order Event
┌─────────────────────────────────────────────────────────────┐
│ Kafka Producer:                                              │
│   Topic: order-events                                         │
│   Partition: hash('order_123456') % 10 = 3                  │
│   Key: 'order_123456'                                        │
│   Message: {                                                  │
│     "event_type": "order.created",                          │
│     "order_id": "order_123456",                             │
│     "user_id": "user_789",                                   │
│     "total": 221.97,                                         │
│     "timestamp": "2024-01-20T15:30:00Z"                     │
│   }                                                          │
│   Latency: < 5ms                                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Consumers Process Event
┌─────────────────────────────────────────────────────────────┐
│ Consumer 1: Notification Service                            │
│ - Receives order.created event                              │
│ - Sends confirmation email to user_789                      │
│                                                              │
│ Consumer 2: Analytics Service                               │
│ - Receives order.created event                              │
│ - Updates order metrics                                     │
│                                                              │
│ Consumer 3: Inventory Service                               │
│ - Receives order.created event                              │
│ - Confirms inventory reservation                            │
│ - Updates stock levels                                      │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Payment Processing Event**

```
Scenario: Payment service crashes after processing payment

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Payment Service processes payment 'pay_abc123'        │
│ T+1s: Payment succeeds, publishes payment.succeeded event  │
│ T+2s: Payment Service crashes (OOM)                         │
│ T+2s: Order creation not completed                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+5s: Saga Orchestrator detects timeout                    │
│ - Payment succeeded but order not created                   │
│ - Triggers compensation: Create order with existing payment│
│                                                              │
│ T+6s: Order Service receives compensation event            │
│ - Creates order 'order_123456'                              │
│ - Links to existing payment 'pay_abc123'                    │
│ - Publishes order.created event                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Caching (Redis)

### A) CONCEPT: What is Redis Caching in Checkout Context?

Redis provides critical caching for checkout operations:

1. **Cart Cache**: Fast cart access, TTL for expiration
2. **Inventory Cache**: Product availability and pricing
3. **Payment Status Cache**: Payment processing status
4. **Order Cache**: Recent orders for quick access

### B) OUR USAGE: How We Use Redis Here

**Cache Types:**

| Cache | Key | TTL | Purpose |
|-------|-----|-----|---------|
| Cart | `cart:{cart_id}` | 30 minutes | Active cart data |
| Inventory | `inventory:{product_id}` | 5 minutes | Product stock and price |
| Payment Status | `payment:{payment_id}` | 1 hour | Payment processing status |
| Order | `order:{order_id}` | 24 hours | Recent order data |

**Cart Caching:**

```java
@Service
public class CartCacheService {
    
    private final RedisTemplate<String, Cart> redis;
    private static final Duration TTL = Duration.ofMinutes(30);
    
    public Cart getCart(String cartId) {
        String key = "cart:" + cartId;
        Cart cached = redis.opsForValue().get(key);
        
        if (cached != null) {
            return cached;
        }
        
        // Load from database
        Cart cart = cartRepository.findById(cartId);
        
        if (cart != null && cart.getStatus() == CartStatus.ACTIVE) {
            // Cache active carts only
            redis.opsForValue().set(key, cart, TTL);
        }
        
        return cart;
    }
    
    public void updateCart(String cartId, Cart cart) {
        String key = "cart:" + cartId;
        redis.opsForValue().set(key, cart, TTL);
    }
}
```

**Inventory Caching:**

```java
@Service
public class InventoryCacheService {
    
    private final RedisTemplate<String, ProductInventory> redis;
    private static final Duration TTL = Duration.ofMinutes(5);
    
    public ProductInventory getInventory(String productId) {
        String key = "inventory:" + productId;
        ProductInventory cached = redis.opsForValue().get(key);
        
        if (cached != null) {
            return cached;
        }
        
        // Load from database
        ProductInventory inventory = inventoryRepository.findByProductId(productId);
        
        if (inventory != null) {
            redis.opsForValue().set(key, inventory, TTL);
        }
        
        return inventory;
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Redis Technology-Level

**Normal Flow: Cart Operations**

```
Step 1: User Adds Item to Cart
┌─────────────────────────────────────────────────────────────┐
│ Request: POST /v1/cart/items                                │
│ Body: { "product_id": "prod_123", "quantity": 2 }          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Cart Cache
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET cart:cart_abc123                                  │
│ Result: HIT (cached cart exists)                            │
│ Value: {                                                      │
│   "cart_id": "cart_abc123",                                 │
│   "items": [{"product_id": "prod_456", "quantity": 1}],    │
│   "total": 49.99                                             │
│ }                                                            │
│ Latency: ~1ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Check Inventory Cache
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET inventory:prod_123                               │
│ Result: HIT                                                  │
│ Value: {                                                     │
│   "product_id": "prod_123",                                 │
│   "available_quantity": 10,                                │
│   "price": 99.99                                            │
│ }                                                            │
│ Latency: ~1ms                                                │
│ Decision: Product available, proceed                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Update Cart
┌─────────────────────────────────────────────────────────────┐
│ Cart Service:                                                │
│ - Add item to cart                                           │
│ - Calculate new total: $249.97                               │
│ - Update Redis: SET cart:cart_abc123 {...} EX 1800          │
│ - Update PostgreSQL (async)                                  │
│ Latency: ~2ms                                                │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Cache Miss**

```
Scenario: Cart cache expired, need to reload from database

┌─────────────────────────────────────────────────────────────┐
│ T+0ms: Redis: GET cart:cart_abc123 → MISS                   │
│ T+1ms: Query PostgreSQL: SELECT * FROM carts WHERE cart_id = 'cart_abc123'│
│ T+10ms: Database returns cart data                          │
│ T+11ms: Cache cart in Redis: SET cart:cart_abc123 {...} EX 1800│
│ T+12ms: Return cart to user                                 │
│ Total latency: ~12ms (vs ~1ms with cache)                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Saga Pattern for Distributed Transactions

### A) CONCEPT: What is the Saga Pattern?

The Saga pattern manages distributed transactions across multiple services without using two-phase commit (2PC). Instead, it uses a sequence of local transactions with compensation actions.

**Why Saga Pattern?**
- 2PC is blocking and doesn't scale
- Saga pattern is non-blocking and scalable
- Each service maintains its own database
- Compensation handles failures

### B) OUR USAGE: How We Use Saga Pattern Here

**Order Creation Saga:**

```java
@Service
public class OrderSagaOrchestrator {
    
    public OrderResult createOrder(CreateOrderRequest request) {
        String sagaId = UUID.randomUUID().toString();
        
        try {
            // Step 1: Reserve Inventory
            ReservationResult reservation = inventoryService.reserveInventory(
                request.getItems(), sagaId
            );
            
            if (!reservation.isSuccess()) {
                return OrderResult.failed("Insufficient inventory");
            }
            
            // Step 2: Process Payment
            PaymentResult payment = paymentService.processPayment(
                request.getPaymentMethod(), request.getAmount(), sagaId
            );
            
            if (!payment.isSuccess()) {
                // Compensate: Release inventory
                inventoryService.releaseReservation(reservation.getReservationId());
                return OrderResult.failed("Payment failed");
            }
            
            // Step 3: Create Order
            Order order = orderService.createOrder(
                request, reservation.getReservationId(), payment.getPaymentId(), sagaId
            );
            
            // Step 4: Confirm Reservation
            inventoryService.confirmReservation(reservation.getReservationId());
            
            return OrderResult.success(order);
            
        } catch (Exception e) {
            // Compensate all steps
            compensate(sagaId);
            return OrderResult.failed("Order creation failed");
        }
    }
    
    private void compensate(String sagaId) {
        // Release inventory reservations
        // Refund payments
        // Clean up partial orders
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Saga Pattern

**Normal Flow: Order Creation**

```
Step 1: Reserve Inventory
┌─────────────────────────────────────────────────────────────┐
│ Saga Orchestrator → Inventory Service                      │
│ Request: Reserve 2x prod_123, 1x prod_456                  │
│                                                              │
│ Inventory Service:                                          │
│ - Check availability: prod_123 (10 available), prod_456 (5 available)│
│ - Create reservations: res_1, res_2                        │
│ - Update reserved_quantity in database                      │
│ - Return reservation_id: res_123                           │
│ Latency: ~50ms                                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Process Payment
┌─────────────────────────────────────────────────────────────┐
│ Saga Orchestrator → Payment Service                        │
│ Request: Process $221.97 payment                           │
│                                                              │
│ Payment Service:                                             │
│ - Validate payment method                                    │
│ - Call payment gateway                                       │
│ - Gateway processes payment (2 seconds)                    │
│ - Store payment record in database                          │
│ - Return payment_id: pay_abc123                             │
│ Latency: ~2 seconds                                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Create Order
┌─────────────────────────────────────────────────────────────┐
│ Saga Orchestrator → Order Service                          │
│ Request: Create order with reservation and payment         │
│                                                              │
│ Order Service:                                               │
│ - Create order record in database                           │
│ - Link to reservation_id and payment_id                     │
│ - Publish order.created event                               │
│ - Return order_id: order_123456                             │
│ Latency: ~20ms                                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Confirm Reservation
┌─────────────────────────────────────────────────────────────┐
│ Saga Orchestrator → Inventory Service                      │
│ Request: Confirm reservation res_123                       │
│                                                              │
│ Inventory Service:                                           │
│ - Update reservation status to 'confirmed'                 │
│ - Update stock_quantity (deduct reserved)                    │
│ - Release reserved_quantity                                 │
│ Latency: ~30ms                                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Success: Order Created
Total latency: ~2.1 seconds
```

**Failure Flow: Payment Failure**

```
Step 1: Reserve Inventory (Success)
┌─────────────────────────────────────────────────────────────┐
│ Inventory reserved: res_123                                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Process Payment (Failure)
┌─────────────────────────────────────────────────────────────┐
│ Payment Service:                                             │
│ - Payment gateway returns: "Card declined"                 │
│ - Payment status: 'failed'                                  │
│ - Return failure to orchestrator                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Compensation
┌─────────────────────────────────────────────────────────────┐
│ Saga Orchestrator:                                           │
│ - Detects payment failure                                    │
│ - Triggers compensation: Release inventory reservation      │
│                                                              │
│ Inventory Service:                                           │
│ - Release reservation res_123                               │
│ - Update reserved_quantity (decrement)                     │
│ - Reservation status: 'released'                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Result: Order creation failed, inventory released
No partial state left in system
```

---

## Summary

| Component | Technology | Key Configuration |
|-----------|------------|-------------------|
| Event Streaming | Kafka | 10 partitions, order_id key |
| Cart Cache | Redis | 30-minute TTL |
| Inventory Cache | Redis | 5-minute TTL |
| Payment Cache | Redis | 1-hour TTL |
| Saga Pattern | Custom orchestrator | Compensation on failures |
| Transaction Management | Saga pattern | No 2PC, compensation-based |


