# E-commerce Checkout - API & Schema Design

## API Design Philosophy

E-commerce checkout APIs prioritize:

1. **Security**: PCI-DSS compliance for payment data
2. **Idempotency**: Prevent duplicate orders and payments
3. **Reliability**: Handle failures gracefully
4. **Performance**: Fast checkout completion
5. **User Experience**: Clear error messages and retry mechanisms

---

## Base URL Structure

```
REST API:     https://api.ecommerce.com/v1
Internal API: https://checkout.internal/api/v1
```

---

## API Versioning Strategy

We use URL path versioning (`/v1/`, `/v2/`) because:
- Easy to understand and implement
- Clear in logs and documentation
- Allows running multiple versions simultaneously

**Backward Compatibility Rules:**

Non-breaking changes (no version bump):
- Adding new optional fields
- Adding new endpoints
- Adding new error codes

Breaking changes (require new version):
- Removing fields
- Changing field types
- Changing endpoint paths

**Deprecation Policy:**
1. Announce deprecation 6 months in advance
2. Return Deprecation header
3. Maintain old version for 12 months after new version release

---

## Authentication

### JWT Token Authentication

```http
POST /v1/cart
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

**Guest Checkout:**
- Guest users can checkout without authentication
- Session-based cart management
- Limited to single-device access

---

## Rate Limiting Headers

Every response includes rate limit information:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
```

**Rate Limits:**

| Endpoint Type | Requests/minute |
|---------------|-----------------|
| Cart Operations | 300 |
| Checkout | 60 |
| Payment | 30 |
| Order History | 1000 |

---

## Error Model

All error responses follow this standard envelope structure:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "field": "field_name",
      "reason": "Specific reason"
    },
    "request_id": "req_123456"
  }
}
```

**Error Codes Reference:**

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | INVALID_INPUT | Request validation failed |
| 400 | INVALID_CART | Cart is empty or invalid |
| 400 | INSUFFICIENT_INVENTORY | Product out of stock |
| 400 | INVALID_PAYMENT | Payment information invalid |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Cart or order not found |
| 409 | CONFLICT | Order already exists |
| 422 | PAYMENT_FAILED | Payment processing failed |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

**Error Response Examples:**

```json
// 400 Bad Request - Insufficient inventory
{
  "error": {
    "code": "INSUFFICIENT_INVENTORY",
    "message": "Product is out of stock",
    "details": {
      "field": "product_id",
      "reason": "Product 'prod_123' has 0 units available, requested 2"
    },
    "request_id": "req_abc123"
  }
}

// 422 Payment Failed
{
  "error": {
    "code": "PAYMENT_FAILED",
    "message": "Payment processing failed",
    "details": {
      "field": "payment",
      "reason": "Card declined: Insufficient funds",
      "payment_id": "pay_xyz789"
    },
    "request_id": "req_def456"
  }
}
```

---

## Idempotency Implementation

**Idempotency-Key Header:**

Clients must include `Idempotency-Key` header for all write operations:
```http
Idempotency-Key: <uuid-v4>
```

**Deduplication Storage:**

Idempotency keys are stored in Redis with 24-hour TTL:
```
Key: idempotency:{key}
Value: Serialized response
TTL: 24 hours
```

**Retry Semantics:**

1. Client sends request with Idempotency-Key
2. Server checks Redis for existing key
3. If found: Return cached response (same status code + body)
4. If not found: Process request, cache response, return result
5. Retries with same key within 24 hours return cached response

**Per-Endpoint Idempotency:**

| Endpoint | Idempotent? | Mechanism |
|----------|-------------|-----------|
| POST /v1/cart/items | Yes | Idempotency-Key header |
| POST /v1/checkout | Yes | Idempotency-Key header |
| POST /v1/payments | Yes | Idempotency-Key header (critical) |
| POST /v1/orders | Yes | Idempotency-Key header |

---

## Core API Endpoints

### 1. Cart Management

#### Add Item to Cart

**Endpoint:** `POST /v1/cart/items`

**Request:**

```http
POST /v1/cart/items HTTP/1.1
Host: api.ecommerce.com
Authorization: Bearer user_token
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

{
  "product_id": "prod_123",
  "quantity": 2,
  "variant_id": "var_456"
}
```

**Response (200 OK):**

```json
{
  "cart": {
    "cart_id": "cart_abc123",
    "items": [
      {
        "item_id": "item_xyz",
        "product_id": "prod_123",
        "product_name": "Wireless Headphones",
        "quantity": 2,
        "unit_price": 99.99,
        "total_price": 199.98,
        "image_url": "https://cdn.ecommerce.com/products/prod_123.jpg"
      }
    ],
    "subtotal": 199.98,
    "tax": 16.00,
    "shipping": 5.99,
    "total": 221.97,
    "currency": "USD"
  }
}
```

#### Get Cart

**Endpoint:** `GET /v1/cart`

**Response (200 OK):**

```json
{
  "cart": {
    "cart_id": "cart_abc123",
    "items": [...],
    "subtotal": 199.98,
    "tax": 16.00,
    "shipping": 5.99,
    "total": 221.97,
    "currency": "USD",
    "expires_at": "2024-01-20T16:00:00Z"
  }
}
```

#### Update Cart Item

**Endpoint:** `PUT /v1/cart/items/{item_id}`

**Request:**

```json
{
  "quantity": 3
}
```

#### Remove Cart Item

**Endpoint:** `DELETE /v1/cart/items/{item_id}`

---

### 2. Checkout

#### Start Checkout

**Endpoint:** `POST /v1/checkout`

**Request:**

```http
POST /v1/checkout HTTP/1.1
Host: api.ecommerce.com
Authorization: Bearer user_token
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440001

{
  "cart_id": "cart_abc123",
  "shipping_address": {
    "name": "John Doe",
    "street": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "zip": "94102",
    "country": "US"
  },
  "shipping_method": "standard",
  "billing_address": {
    "same_as_shipping": true
  }
}
```

**Response (200 OK):**

```json
{
  "checkout_id": "checkout_xyz789",
  "cart": {...},
  "shipping_address": {...},
  "shipping_cost": 5.99,
  "estimated_delivery": "2024-01-25",
  "reservation_expires_at": "2024-01-20T15:35:00Z"
}
```

**Error Response (400 Bad Request):**

```json
{
  "error": {
    "code": "INSUFFICIENT_INVENTORY",
    "message": "Some items are no longer available",
    "details": {
      "unavailable_items": [
        {
          "product_id": "prod_123",
          "requested": 2,
          "available": 1
        }
      ]
    }
  }
}
```

---

### 3. Payment Processing

#### Process Payment

**Endpoint:** `POST /v1/payments`

**Request:**

```http
POST /v1/payments HTTP/1.1
Host: api.ecommerce.com
Authorization: Bearer user_token
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440002

{
  "checkout_id": "checkout_xyz789",
  "payment_method": {
    "type": "credit_card",
    "card_number": "4111111111111111",
    "expiry_month": 12,
    "expiry_year": 2025,
    "cvv": "123",
    "cardholder_name": "John Doe"
  },
  "amount": 221.97,
  "currency": "USD"
}
```

**Response (200 OK):**

```json
{
  "payment_id": "pay_abc123",
  "checkout_id": "checkout_xyz789",
  "status": "succeeded",
  "amount": 221.97,
  "currency": "USD",
  "payment_method": {
    "type": "credit_card",
    "last4": "1111",
    "brand": "visa"
  },
  "processed_at": "2024-01-20T15:30:00Z"
}
```

**Error Response (422 Payment Failed):**

```json
{
  "error": {
    "code": "PAYMENT_FAILED",
    "message": "Payment processing failed",
    "details": {
      "reason": "Card declined: Insufficient funds",
      "payment_id": "pay_xyz789",
      "retryable": false
    }
  }
}
```

---

### 4. Order Creation

#### Create Order

**Endpoint:** `POST /v1/orders`

**Request:**

```http
POST /v1/orders HTTP/1.1
Host: api.ecommerce.com
Authorization: Bearer user_token
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440003

{
  "checkout_id": "checkout_xyz789",
  "payment_id": "pay_abc123"
}
```

**Response (201 Created):**

```json
{
  "order": {
    "order_id": "order_123456",
    "user_id": "user_789",
    "status": "confirmed",
    "items": [
      {
        "product_id": "prod_123",
        "product_name": "Wireless Headphones",
        "quantity": 2,
        "unit_price": 99.99,
        "total_price": 199.98
      }
    ],
    "subtotal": 199.98,
    "tax": 16.00,
    "shipping": 5.99,
    "total": 221.97,
    "currency": "USD",
    "shipping_address": {...},
    "payment_method": {
      "type": "credit_card",
      "last4": "1111"
    },
    "created_at": "2024-01-20T15:30:00Z",
    "estimated_delivery": "2024-01-25"
  }
}
```

---

### 5. Order Management

#### Get Order

**Endpoint:** `GET /v1/orders/{order_id}`

**Response (200 OK):**

```json
{
  "order": {
    "order_id": "order_123456",
    "status": "shipped",
    "items": [...],
    "total": 221.97,
    "tracking_number": "1Z999AA10123456784",
    "shipped_at": "2024-01-22T10:00:00Z",
    "estimated_delivery": "2024-01-25"
  }
}
```

#### Get Order History

**Endpoint:** `GET /v1/orders`

**Query Parameters:**
- `cursor`: Pagination cursor
- `limit`: Number of results (default: 20, max: 100)
- `status`: Filter by status

**Response (200 OK):**

```json
{
  "orders": [
    {
      "order_id": "order_123456",
      "status": "delivered",
      "total": 221.97,
      "created_at": "2024-01-20T15:30:00Z"
    }
  ],
  "pagination": {
    "next_cursor": "cursor_xyz",
    "has_more": true
  }
}
```

#### Cancel Order

**Endpoint:** `POST /v1/orders/{order_id}/cancel`

**Request:**

```json
{
  "reason": "Changed my mind"
}
```

**Response (200 OK):**

```json
{
  "order_id": "order_123456",
  "status": "cancelled",
  "cancelled_at": "2024-01-20T16:00:00Z",
  "refund_status": "pending"
}
```

---

## Pagination Strategy

We use **cursor-based pagination** for order history:

**Why Cursor over Offset?**
- Offset pagination becomes slow with large datasets
- Cursor pagination is consistent even with concurrent writes
- Better performance for large result sets

**Implementation:**

```java
public class CursorPagination {
    private String cursor;  // Base64 encoded: timestamp + order_id
    private int limit;      // Max 100
    
    public String encodeCursor(LocalDateTime timestamp, String orderId) {
        String data = timestamp.toString() + ":" + orderId;
        return Base64.getEncoder().encodeToString(data.getBytes());
    }
    
    public Cursor decodeCursor(String cursor) {
        String data = new String(Base64.getDecoder().decode(cursor));
        String[] parts = data.split(":");
        return new Cursor(LocalDateTime.parse(parts[0]), parts[1]);
    }
}
```

---

## Database Schema Design

### Cart Schema

```sql
CREATE TABLE carts (
    cart_id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50),  -- NULL for guest carts
    session_id VARCHAR(100),  -- For guest carts
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE,
    
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

### Order Schema

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

### Inventory Reservation Schema

```sql
CREATE TABLE inventory_reservations (
    reservation_id VARCHAR(50) PRIMARY KEY,
    product_id VARCHAR(50) NOT NULL,
    cart_id VARCHAR(50) NOT NULL,
    checkout_id VARCHAR(50),
    quantity INTEGER NOT NULL,
    status VARCHAR(20) DEFAULT 'reserved',
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    released_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_status CHECK (status IN ('reserved', 'confirmed', 'released', 'expired'))
);

CREATE INDEX idx_reservations_product ON inventory_reservations(product_id, status);
CREATE INDEX idx_reservations_cart ON inventory_reservations(cart_id);
CREATE INDEX idx_reservations_checkout ON inventory_reservations(checkout_id);
CREATE INDEX idx_reservations_expires ON inventory_reservations(expires_at) WHERE status = 'reserved';
```

### Payment Schema

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
```

---

## Summary

| Component | Technology/Approach |
|-----------|-------------------|
| API Versioning | URL path versioning (/v1/) |
| Authentication | JWT tokens + guest sessions |
| Idempotency | Idempotency-Key header + Redis |
| Pagination | Cursor-based |
| Error Handling | Standard error envelope |
| Cart Storage | Redis (active) + PostgreSQL (history) |
| Order Storage | PostgreSQL |
| Payment Storage | PostgreSQL (encrypted) |


