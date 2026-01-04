# E-commerce Checkout - Data Model & Architecture

## Component Overview

Before looking at diagrams, let's understand each component and why it exists.

### Components Explained

| Component | Purpose | Why It Exists |
|-----------|---------|---------------|
| **Cart Service** | Manages shopping cart state | Users need to collect items before checkout |
| **Inventory Service** | Manages product availability | Prevents overselling, reserves inventory |
| **Payment Service** | Processes payments | Secure payment handling, gateway integration |
| **Order Service** | Creates and manages orders | Order lifecycle management |
| **Shipping Service** | Calculates shipping costs | Shipping cost calculation and address validation |
| **Notification Service** | Sends confirmations | User communication for order status |

---

## Database Choices

| Data Type | Database | Rationale |
|-----------|----------|-----------|
| Cart (active) | Redis | Fast access, TTL support for expiration |
| Cart (history) | PostgreSQL | Persistent storage, analytics |
| Orders | PostgreSQL | ACID transactions, complex queries |
| Inventory | PostgreSQL | Strong consistency required |
| Inventory Reservations | PostgreSQL | ACID for reservation management |
| Payments | PostgreSQL | ACID, audit trail, compliance |
| Payment Gateway Cache | Redis | Fast lookup of payment status |

---

## Consistency Model

**CAP Theorem Tradeoff:**

We choose **Consistency + Partition Tolerance (CP)** for critical operations:
- **Consistency**: Inventory and payments must be consistent
- **Partition Tolerance**: System continues operating during network partitions
- **Availability**: Sacrificed (prefer consistency over availability for critical operations)

**Why CP over AP?**
- Cannot oversell inventory (consistency critical)
- Payment must be processed exactly once (consistency critical)
- Order creation must be atomic (consistency critical)
- Better to fail than create inconsistent state

**ACID vs BASE:**

**ACID (Strong Consistency) for:**
- Inventory updates (PostgreSQL, prevents overselling)
- Payment processing (PostgreSQL, prevents duplicate charges)
- Order creation (PostgreSQL, atomic transaction)
- Inventory reservations (PostgreSQL, prevents double reservation)

**BASE (Eventual Consistency) for:**
- Cart state (Redis, acceptable staleness)
- Order status updates (eventual synchronization)
- Inventory cache (may be stale, verified before reservation)

**Per-Operation Consistency Guarantees:**

| Operation | Consistency Level | Guarantee |
|-----------|------------------|-----------|
| Reserve inventory | Strong | Immediately reserved, no overselling |
| Process payment | Strong | Exactly-once processing |
| Create order | Strong | Atomic order creation |
| Update cart | Eventual | Cart may be stale briefly |
| Get order status | Strong | Always current status |

**Saga Pattern for Distributed Transactions:**

Since order creation involves multiple services (inventory, payment, order), we use the Saga pattern:

```
1. Reserve Inventory (Inventory Service)
2. Process Payment (Payment Service)
3. Create Order (Order Service)
4. If any step fails, compensate (release inventory, refund payment)
```

---

## Cart Schema

### Redis Structure (Active Carts)

```
# Cart data (Hash)
HSET cart:{cart_id} user_id user_123 session_id sess_456 status active expires_at 1705764600

# Cart items (Hash)
HSET cart:{cart_id}:items item_1 '{"product_id":"prod_123","quantity":2,"price":99.99}'
HSET cart:{cart_id}:items item_2 '{"product_id":"prod_456","quantity":1,"price":49.99}'

# Cart TTL
EXPIRE cart:{cart_id} 1800  # 30 minutes
```

### PostgreSQL Schema (Cart History)

```sql
CREATE TABLE carts (
    cart_id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50),
    session_id VARCHAR(100),
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE,
    converted_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_status CHECK (status IN ('active', 'abandoned', 'converted'))
);

CREATE INDEX idx_carts_user ON carts(user_id);
CREATE INDEX idx_carts_session ON carts(session_id);
CREATE INDEX idx_carts_expires ON carts(expires_at) WHERE status = 'active';

CREATE TABLE cart_items (
    item_id VARCHAR(50) PRIMARY KEY,
    cart_id VARCHAR(50) NOT NULL,
    product_id VARCHAR(50) NOT NULL,
    variant_id VARCHAR(50),
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CONSTRAINT fk_cart FOREIGN KEY (cart_id) REFERENCES carts(cart_id) ON DELETE CASCADE,
    CONSTRAINT valid_quantity CHECK (quantity > 0)
);

CREATE INDEX idx_cart_items_cart ON cart_items(cart_id);
CREATE INDEX idx_cart_items_product ON cart_items(product_id);
```

---

## Order Schema

```sql
CREATE TABLE orders (
    order_id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL,
    checkout_id VARCHAR(50) UNIQUE,
    payment_id VARCHAR(50) UNIQUE,
    
    status VARCHAR(20) DEFAULT 'pending',
    
    -- Pricing
    subtotal DECIMAL(10, 2) NOT NULL,
    tax DECIMAL(10, 2) NOT NULL,
    shipping_cost DECIMAL(10, 2) NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    
    -- Shipping
    shipping_address JSONB NOT NULL,
    shipping_method VARCHAR(50),
    tracking_number VARCHAR(100),
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    confirmed_at TIMESTAMP WITH TIME ZONE,
    shipped_at TIMESTAMP WITH TIME ZONE,
    delivered_at TIMESTAMP WITH TIME ZONE,
    cancelled_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_status CHECK (status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled'))
);

CREATE INDEX idx_orders_user ON orders(user_id, created_at DESC);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_checkout ON orders(checkout_id);
CREATE INDEX idx_orders_payment ON orders(payment_id);

CREATE TABLE order_items (
    item_id VARCHAR(50) PRIMARY KEY,
    order_id VARCHAR(50) NOT NULL,
    product_id VARCHAR(50) NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    variant_id VARCHAR(50),
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    total_price DECIMAL(10, 2) NOT NULL,
    
    CONSTRAINT fk_order FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE
);

CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);
```

---

## Inventory Schema

```sql
CREATE TABLE products (
    product_id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    sku VARCHAR(100) UNIQUE,
    price DECIMAL(10, 2) NOT NULL,
    stock_quantity INTEGER DEFAULT 0,
    reserved_quantity INTEGER DEFAULT 0,
    available_quantity INTEGER GENERATED ALWAYS AS (stock_quantity - reserved_quantity) STORED,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_stock ON products(available_quantity) WHERE available_quantity > 0;

CREATE TABLE inventory_reservations (
    reservation_id VARCHAR(50) PRIMARY KEY,
    product_id VARCHAR(50) NOT NULL,
    cart_id VARCHAR(50),
    checkout_id VARCHAR(50),
    order_id VARCHAR(50),
    quantity INTEGER NOT NULL,
    status VARCHAR(20) DEFAULT 'reserved',
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    released_at TIMESTAMP WITH TIME ZONE,
    confirmed_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT fk_product FOREIGN KEY (product_id) REFERENCES products(product_id),
    CONSTRAINT valid_status CHECK (status IN ('reserved', 'confirmed', 'released', 'expired'))
);

CREATE INDEX idx_reservations_product ON inventory_reservations(product_id, status);
CREATE INDEX idx_reservations_cart ON inventory_reservations(cart_id);
CREATE INDEX idx_reservations_checkout ON inventory_reservations(checkout_id);
CREATE INDEX idx_reservations_expires ON inventory_reservations(expires_at) WHERE status = 'reserved';
```

---

## Payment Schema

```sql
CREATE TABLE payments (
    payment_id VARCHAR(50) PRIMARY KEY,
    checkout_id VARCHAR(50) NOT NULL,
    order_id VARCHAR(50),
    
    amount DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    
    payment_method_type VARCHAR(50) NOT NULL,
    payment_method_details JSONB,  -- Encrypted card details
    
    status VARCHAR(20) DEFAULT 'pending',
    gateway_transaction_id VARCHAR(100),
    gateway_response JSONB,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    processed_at TIMESTAMP WITH TIME ZONE,
    failed_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_status CHECK (status IN ('pending', 'processing', 'succeeded', 'failed', 'refunded'))
);

CREATE INDEX idx_payments_checkout ON payments(checkout_id);
CREATE INDEX idx_payments_order ON payments(order_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_gateway ON payments(gateway_transaction_id);
```

---

## Entity Relationship Diagram

```
┌─────────────────────┐       ┌─────────────────────┐
│       carts         │       │       orders        │
├─────────────────────┤       ├─────────────────────┤
│ cart_id (PK)        │       │ order_id (PK)       │
│ user_id             │       │ user_id             │
│ session_id          │       │ checkout_id (FK)    │
│ status              │       │ payment_id (FK)     │
│ expires_at          │       │ status              │
└─────────────────────┘       │ total               │
         │                     └─────────────────────┘
         │                             │
         │ 1:N                         │ 1:N
         ▼                             ▼
┌─────────────────────┐       ┌─────────────────────┐
│    cart_items       │       │    order_items       │
├─────────────────────┤       ├─────────────────────┤
│ item_id (PK)        │       │ item_id (PK)         │
│ cart_id (FK)        │       │ order_id (FK)        │
│ product_id          │       │ product_id           │
│ quantity            │       │ quantity             │
│ unit_price          │       │ unit_price           │
└─────────────────────┘       └─────────────────────┘
         │                             │
         │                             │
         └─────────────┬───────────────┘
                       │
                       ▼
         ┌─────────────────────┐
         │     products        │
         ├─────────────────────┤
         │ product_id (PK)     │
         │ name                │
         │ stock_quantity      │
         │ reserved_quantity   │
         └─────────────────────┘
                       │
                       │ 1:N
                       ▼
         ┌─────────────────────┐
         │ inventory_reservations│
         ├─────────────────────┤
         │ reservation_id (PK) │
         │ product_id (FK)     │
         │ cart_id             │
         │ checkout_id          │
         │ quantity             │
         │ expires_at           │
         └─────────────────────┘

┌─────────────────────┐
│      payments       │
├─────────────────────┤
│ payment_id (PK)     │
│ checkout_id         │
│ order_id            │
│ amount              │
│ status              │
│ gateway_transaction_id│
└─────────────────────┘
```

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    CLIENTS                                           │
│                    (Web Browser, Mobile App)                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                 API GATEWAY                                          │
│                          (Load Balancer, Rate Limiting)                             │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
                    ▼                   ▼                   ▼
┌──────────────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│    Cart Service          │  │  Checkout Service│  │  Order Service   │
│  - Add/remove items      │  │  - Start checkout  │  │  - Create order  │
│  - Update quantities     │  │  - Validate cart  │  │  - Get order     │
│  - Calculate totals      │  │  - Reserve inventory│ │  - Cancel order │
└──────────────────────────┘  └──────────────────┘  └──────────────────┘
         │                              │                      │
         │                              │                      │
         └──────────────┬───────────────┼──────────────────────┘
                        │               │                      │
                        ▼               ▼                      ▼
┌──────────────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  Inventory Service       │  │  Payment Service │  │  Shipping Service│
│  - Check availability    │  │  - Process payment│  │  - Calculate cost │
│  - Reserve inventory     │  │  - Handle failures│  │  - Validate address│
│  - Release reservation   │  │  - Refund        │  │  - Get options   │
└──────────────────────────┘  └──────────────────┘  └──────────────────┘
         │                              │
         │                              │
         └──────────────┬───────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                  DATA LAYER                                         │
│                                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Redis      │  │  PostgreSQL  │  │     Kafka    │  │  Payment     │          │
│  │  (Cart Cache)│  │  (Orders/     │  │  (Events)    │  │  Gateway     │          │
│  │              │  │   Inventory)  │  │              │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Order Creation Flow (Saga Pattern)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          ORDER CREATION SAGA                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

Step 1: Start Checkout
User → Checkout Service → Inventory Service
  │                          │
  │                          ▼
  │                  Reserve Inventory
  │                  (reservation_id)
  │                          │
  │                          ▼
  │                  Return checkout_id
  │                          │
  ▼                          │
Step 2: Process Payment
User → Payment Service → Payment Gateway
  │                          │
  │                          ▼
  │                  Process Payment
  │                  (payment_id)
  │                          │
  │                          ▼
  │                  Return payment_id
  │                          │
  ▼                          │
Step 3: Create Order
User → Order Service → PostgreSQL
  │                          │
  │                          ▼
  │                  Create Order
  │                  (order_id)
  │                          │
  │                          ▼
  │                  Confirm Reservation
  │                  Update Inventory
  │                          │
  │                          ▼
  │                  Return order_id
  │                          │
  ▼                          │
Success: Order Created

If any step fails:
  - Release inventory reservation
  - Refund payment (if processed)
  - Rollback order creation
```

---

## Failure Handling

```
Failure Type           Detection              Recovery
─────────────────────────────────────────────────────────────────────────────────────

┌─────────────────┐
│ Inventory       │ ─── Reservation ─────── Release reservation
│ Reservation     │     timeout              Retry checkout
│ Timeout         │                          Notify user
└─────────────────┘

┌─────────────────┐
│ Payment         │ ─── Gateway ─────────── Retry payment
│ Failure         │     error               Alternative payment method
│                 │                          Release inventory
└─────────────────┘

┌─────────────────┐
│ Order Creation  │ ─── Database ────────── Retry order creation
│ Failure         │     error               Compensate (refund, release)
│                 │                          Alert operations
└─────────────────┘
```

---

## Summary

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Cart Storage | Redis (active) + PostgreSQL (history) | Fast access + persistence |
| Order Storage | PostgreSQL | ACID transactions |
| Inventory | PostgreSQL | Strong consistency |
| Payment | PostgreSQL | Audit trail, compliance |
| Consistency | Strong for critical operations | Cannot oversell, must be accurate |
| Transaction Pattern | Saga pattern | Distributed transaction management |
| Failure Handling | Compensation | Rollback on failures |


