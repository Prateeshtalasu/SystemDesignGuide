# Food Delivery - API & Schema Design

## API Design Philosophy

Food delivery APIs must handle:
1. **Three-sided marketplace**: Customers, restaurants, delivery partners
2. **Complex state machines**: Orders go through many states
3. **Real-time updates**: Order tracking, delivery location
4. **Search with filters**: Restaurant discovery is critical

---

## Base URL Structure

```
Customer API:    https://api.fooddelivery.com/v1
Restaurant API:  https://restaurant.fooddelivery.com/v1
Partner API:     https://partner.fooddelivery.com/v1
WebSocket:       wss://ws.fooddelivery.com/v1
```

### API Versioning Strategy

We use URL path versioning (`/v1/`, `/v2/`) because:
- Different apps may use different versions
- Clear deprecation path
- Easy to maintain backward compatibility

---

## Authentication

### JWT Token Authentication

```http
GET /v1/orders
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

### Token Structure

```json
{
  "sub": "user_123456",
  "type": "customer",  // customer, restaurant, partner
  "restaurant_id": null,  // Set for restaurant users
  "exp": 1735689600,
  "iat": 1735603200
}
```

### Rate Limiting Headers

Every response includes rate limit information:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
```

### Rate Limiting

| Endpoint Type | Limit | Window |
|---------------|-------|--------|
| Search | 60 | 1 minute |
| Order placement | 10 | 1 minute |
| Location updates | 60 | 1 minute |
| General API | 100 | 1 minute |

---

## Error Model

All error responses follow this standard envelope structure:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "field": "field_name",  // Optional
      "reason": "Specific reason"  // Optional
    },
    "request_id": "req_123456"  // For tracing
  }
}
```

**Error Codes Reference:**

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | INVALID_INPUT | Request validation failed |
| 400 | INVALID_LOCATION | Location coordinates are invalid |
| 400 | RESTAURANT_CLOSED | Restaurant is currently closed |
| 400 | ITEM_UNAVAILABLE | Menu item is unavailable |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Order, restaurant, or resource not found |
| 409 | CONFLICT | Order state conflict |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

**Error Response Examples:**

```json
// 400 Bad Request - Restaurant closed
{
  "error": {
    "code": "RESTAURANT_CLOSED",
    "message": "Restaurant is currently closed",
    "details": {
      "field": "restaurant_id",
      "reason": "Restaurant hours: 11:00 AM - 10:00 PM"
    },
    "request_id": "req_abc123"
  }
}

// 409 Conflict - Order state conflict
{
  "error": {
    "code": "CONFLICT",
    "message": "Order state conflict",
    "details": {
      "field": "order_id",
      "reason": "Order is already being prepared"
    },
    "request_id": "req_xyz789"
  }
}

// 429 Rate Limited
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded. Please try again later.",
    "details": {
      "limit": 10,
      "remaining": 0,
      "reset_at": "2024-01-15T11:00:00Z"
    },
    "request_id": "req_def456"
  }
}
```

---

## Idempotency Implementation

**Idempotency-Key Header:**

Clients must include `Idempotency-Key` header for all write operations (critical for orders):
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
| POST /v1/orders | Yes | Idempotency-Key header (CRITICAL - prevents duplicate orders) |
| PUT /v1/orders/{id} | Yes | Idempotency-Key or version-based |
| POST /v1/orders/{id}/cancel | Yes | Idempotency-Key header |
| POST /v1/orders/{id}/rate | Yes | Idempotency-Key header |

**Implementation Example:**

```java
@PostMapping("/v1/orders")
public ResponseEntity<OrderResponse> createOrder(
        @RequestBody CreateOrderRequest request,
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {
    
    // Check for existing idempotency key (CRITICAL for preventing duplicate orders)
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        String cachedResponse = redisTemplate.opsForValue().get(cacheKey);
        if (cachedResponse != null) {
            OrderResponse response = objectMapper.readValue(cachedResponse, OrderResponse.class);
            return ResponseEntity.status(response.getStatus()).body(response);
        }
    }
    
    // Create order
    Order order = orderService.createOrder(request, idempotencyKey);
    OrderResponse response = OrderResponse.from(order);
    
    // Cache response if idempotency key provided
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        redisTemplate.opsForValue().set(
            cacheKey, 
            objectMapper.writeValueAsString(response),
            Duration.ofHours(24)
        );
    }
    
    return ResponseEntity.status(201).body(response);
}
```

---

## Customer API Endpoints

### 1. Search Restaurants

**Endpoint:** `GET /v1/restaurants`

**Purpose:** Find restaurants near customer

**Request:**

```http
GET /v1/restaurants?lat=37.7749&lng=-122.4194&cuisine=italian&sort=rating&page=1&limit=20 HTTP/1.1
Host: api.fooddelivery.com
Authorization: Bearer {token}
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| lat | float | Yes | Customer latitude |
| lng | float | Yes | Customer longitude |
| cuisine | string | No | Filter by cuisine type |
| sort | string | No | rating, distance, delivery_time |
| min_rating | float | No | Minimum rating (1-5) |
| price_level | int | No | 1-4 ($ to $$$$) |
| page | int | No | Page number (default 1) |
| limit | int | No | Results per page (default 20) |

**Response (200 OK):**

```json
{
  "restaurants": [
    {
      "id": "rest_abc123",
      "name": "Mario's Italian Kitchen",
      "cuisine_types": ["Italian", "Pizza"],
      "rating": 4.7,
      "rating_count": 1250,
      "price_level": 2,
      "delivery_fee": 2.99,
      "delivery_time_minutes": {
        "min": 25,
        "max": 35
      },
      "distance_km": 1.2,
      "photo_url": "https://cdn.fooddelivery.com/restaurants/rest_abc123/cover.jpg",
      "is_open": true,
      "promotions": [
        {
          "type": "PERCENTAGE",
          "value": 20,
          "description": "20% off first order"
        }
      ]
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total_count": 156,
    "has_next": true
  },
  "filters_applied": {
    "cuisine": "italian",
    "sort": "rating"
  }
}
```

---

### 2. Get Restaurant Menu

**Endpoint:** `GET /v1/restaurants/{restaurant_id}/menu`

**Response (200 OK):**

```json
{
  "restaurant_id": "rest_abc123",
  "restaurant_name": "Mario's Italian Kitchen",
  "categories": [
    {
      "id": "cat_appetizers",
      "name": "Appetizers",
      "items": [
        {
          "id": "item_001",
          "name": "Bruschetta",
          "description": "Toasted bread with tomatoes, garlic, and basil",
          "price": 8.99,
          "photo_url": "https://cdn.fooddelivery.com/items/item_001.jpg",
          "is_available": true,
          "prep_time_minutes": 10,
          "customizations": [
            {
              "id": "cust_001",
              "name": "Extra Toppings",
              "type": "MULTI_SELECT",
              "required": false,
              "max_selections": 3,
              "options": [
                {"id": "opt_001", "name": "Extra Cheese", "price": 1.50},
                {"id": "opt_002", "name": "Olives", "price": 0.75}
              ]
            }
          ],
          "dietary_tags": ["vegetarian"],
          "calories": 320
        }
      ]
    },
    {
      "id": "cat_pasta",
      "name": "Pasta",
      "items": [...]
    }
  ],
  "operating_hours": {
    "monday": {"open": "11:00", "close": "22:00"},
    "tuesday": {"open": "11:00", "close": "22:00"},
    ...
  },
  "minimum_order": 15.00,
  "delivery_fee": 2.99
}
```

---

### 3. Place Order

**Endpoint:** `POST /v1/orders`

**Request:**

```http
POST /v1/orders HTTP/1.1
Host: api.fooddelivery.com
Authorization: Bearer {token}
Content-Type: application/json
Idempotency-Key: order_req_xyz789

{
  "restaurant_id": "rest_abc123",
  "items": [
    {
      "item_id": "item_001",
      "quantity": 2,
      "customizations": [
        {
          "customization_id": "cust_001",
          "selected_options": ["opt_001"]
        }
      ],
      "special_instructions": "Extra crispy please"
    },
    {
      "item_id": "item_015",
      "quantity": 1,
      "customizations": []
    }
  ],
  "delivery_address": {
    "address_line1": "123 Main St",
    "address_line2": "Apt 4B",
    "city": "San Francisco",
    "state": "CA",
    "zip_code": "94102",
    "latitude": 37.7849,
    "longitude": -122.4094,
    "delivery_instructions": "Ring doorbell twice"
  },
  "payment_method_id": "pm_card_123",
  "tip_amount": 5.00,
  "promo_code": "FIRST20",
  "scheduled_for": null
}
```

**Response (201 Created):**

```json
{
  "order_id": "order_xyz789",
  "status": "PLACED",
  "restaurant": {
    "id": "rest_abc123",
    "name": "Mario's Italian Kitchen",
    "phone": "+1-555-123-4567"
  },
  "items": [
    {
      "item_id": "item_001",
      "name": "Bruschetta",
      "quantity": 2,
      "unit_price": 8.99,
      "customizations_price": 1.50,
      "subtotal": 20.98
    },
    {
      "item_id": "item_015",
      "name": "Spaghetti Carbonara",
      "quantity": 1,
      "unit_price": 16.99,
      "customizations_price": 0,
      "subtotal": 16.99
    }
  ],
  "pricing": {
    "subtotal": 37.97,
    "discount": -7.59,
    "delivery_fee": 2.99,
    "service_fee": 2.50,
    "tax": 3.04,
    "tip": 5.00,
    "total": 43.91
  },
  "delivery_address": {...},
  "estimated_delivery": {
    "min_minutes": 35,
    "max_minutes": 45,
    "estimated_at": "2024-01-15T19:15:00Z"
  },
  "created_at": "2024-01-15T18:30:00Z"
}
```

**Error Responses:**

```json
// 400 Bad Request - Item unavailable
{
  "error": {
    "code": "ITEM_UNAVAILABLE",
    "message": "One or more items are no longer available",
    "details": {
      "unavailable_items": ["item_015"]
    }
  }
}

// 400 Bad Request - Below minimum order
{
  "error": {
    "code": "BELOW_MINIMUM",
    "message": "Order total is below restaurant minimum",
    "details": {
      "minimum": 15.00,
      "current": 12.50
    }
  }
}

// 400 Bad Request - Restaurant closed
{
  "error": {
    "code": "RESTAURANT_CLOSED",
    "message": "Restaurant is currently closed",
    "details": {
      "opens_at": "11:00",
      "timezone": "America/Los_Angeles"
    }
  }
}
```

---

### 4. Get Order Status

**Endpoint:** `GET /v1/orders/{order_id}`

**Response (200 OK):**

```json
{
  "order_id": "order_xyz789",
  "status": "PICKED_UP",
  "status_history": [
    {"status": "PLACED", "timestamp": "2024-01-15T18:30:00Z"},
    {"status": "CONFIRMED", "timestamp": "2024-01-15T18:32:00Z"},
    {"status": "PREPARING", "timestamp": "2024-01-15T18:33:00Z"},
    {"status": "READY", "timestamp": "2024-01-15T18:50:00Z"},
    {"status": "PICKED_UP", "timestamp": "2024-01-15T18:55:00Z"}
  ],
  "restaurant": {
    "id": "rest_abc123",
    "name": "Mario's Italian Kitchen",
    "location": {
      "latitude": 37.7749,
      "longitude": -122.4194
    }
  },
  "delivery_partner": {
    "id": "partner_456",
    "name": "John D.",
    "photo_url": "https://cdn.fooddelivery.com/partners/partner_456.jpg",
    "phone_masked": "***-***-7890",
    "rating": 4.9,
    "vehicle": "Black Honda Civic",
    "current_location": {
      "latitude": 37.7800,
      "longitude": -122.4150,
      "updated_at": "2024-01-15T18:56:30Z"
    }
  },
  "estimated_delivery": {
    "min_minutes": 8,
    "max_minutes": 12,
    "estimated_at": "2024-01-15T19:05:00Z"
  },
  "delivery_address": {...},
  "pricing": {...}
}
```

---

### 5. Cancel Order

**Endpoint:** `POST /v1/orders/{order_id}/cancel`

**Request:**

```json
{
  "reason": "CHANGED_MIND",
  "comment": "Found something else to eat"
}
```

**Response (200 OK):**

```json
{
  "order_id": "order_xyz789",
  "status": "CANCELLED",
  "refund": {
    "amount": 43.91,
    "status": "PROCESSING",
    "estimated_days": 3
  },
  "cancellation_fee": 0.00,
  "cancelled_at": "2024-01-15T18:31:00Z"
}
```

**Cancellation Fee Rules:**

| Order Status | Fee |
|--------------|-----|
| PLACED (within 1 min) | $0 |
| PLACED (after 1 min) | $0 |
| CONFIRMED | $2 |
| PREPARING | $5 + 50% of order |
| READY | Full order value |
| PICKED_UP | No cancellation allowed |

---

## Restaurant API Endpoints

### 6. Get Incoming Orders

**Endpoint:** `GET /v1/restaurant/orders?status=PLACED`

**Response (200 OK):**

```json
{
  "orders": [
    {
      "order_id": "order_xyz789",
      "status": "PLACED",
      "customer": {
        "name": "Jane S.",
        "order_count": 15
      },
      "items": [
        {
          "name": "Bruschetta",
          "quantity": 2,
          "customizations": ["Extra Cheese"],
          "special_instructions": "Extra crispy please"
        }
      ],
      "total": 43.91,
      "placed_at": "2024-01-15T18:30:00Z",
      "accept_deadline": "2024-01-15T18:35:00Z"
    }
  ],
  "pending_count": 3
}
```

---

### 7. Accept/Reject Order

**Endpoint:** `POST /v1/restaurant/orders/{order_id}/respond`

**Request:**

```json
{
  "action": "ACCEPT",
  "estimated_prep_time_minutes": 20
}
```

**Or for rejection:**

```json
{
  "action": "REJECT",
  "reason": "TOO_BUSY",
  "message": "Kitchen is backed up, try again in 30 minutes"
}
```

---

### 8. Update Order Status

**Endpoint:** `POST /v1/restaurant/orders/{order_id}/status`

**Request:**

```json
{
  "status": "READY"
}
```

**Valid Status Transitions (Restaurant):**

| Current | Allowed Next |
|---------|--------------|
| CONFIRMED | PREPARING |
| PREPARING | READY |

---

### 9. Update Menu Item Availability

**Endpoint:** `PATCH /v1/restaurant/menu/items/{item_id}`

**Request:**

```json
{
  "is_available": false,
  "unavailable_until": "2024-01-15T22:00:00Z"
}
```

---

## Delivery Partner API Endpoints

### 10. Get Available Deliveries

**Endpoint:** `GET /v1/partner/deliveries/available`

**Response (200 OK):**

```json
{
  "deliveries": [
    {
      "delivery_id": "del_abc123",
      "order_id": "order_xyz789",
      "restaurant": {
        "name": "Mario's Italian Kitchen",
        "address": "456 Restaurant Row",
        "location": {
          "latitude": 37.7749,
          "longitude": -122.4194
        }
      },
      "customer": {
        "address": "123 Main St, Apt 4B",
        "location": {
          "latitude": 37.7849,
          "longitude": -122.4094
        }
      },
      "estimated_earnings": 8.50,
      "distance_km": 2.5,
      "expires_at": "2024-01-15T18:52:00Z"
    }
  ]
}
```

---

### 11. Accept Delivery

**Endpoint:** `POST /v1/partner/deliveries/{delivery_id}/accept`

**Response (200 OK):**

```json
{
  "delivery_id": "del_abc123",
  "status": "ACCEPTED",
  "restaurant": {
    "name": "Mario's Italian Kitchen",
    "address": "456 Restaurant Row",
    "phone": "+1-555-123-4567",
    "location": {...}
  },
  "order_details": {
    "items_count": 3,
    "special_instructions": "Fragile items"
  },
  "customer": {
    "name": "Jane S.",
    "address": "123 Main St, Apt 4B",
    "phone_masked": "***-***-1234",
    "delivery_instructions": "Ring doorbell twice"
  },
  "navigation_url": "https://maps.google.com/..."
}
```

---

### 12. Update Delivery Status

**Endpoint:** `POST /v1/partner/deliveries/{delivery_id}/status`

**Request:**

```json
{
  "status": "PICKED_UP",
  "location": {
    "latitude": 37.7749,
    "longitude": -122.4194
  }
}
```

**Valid Status Transitions (Partner):**

| Current | Allowed Next |
|---------|--------------|
| ACCEPTED | ARRIVED_AT_RESTAURANT |
| ARRIVED_AT_RESTAURANT | PICKED_UP |
| PICKED_UP | DELIVERED |

---

## WebSocket API (Real-Time Updates)

### Connection

```javascript
const ws = new WebSocket('wss://ws.fooddelivery.com/v1?token={jwt_token}');
```

### Message Types

**1. Order Status Update (Server → Customer):**

```json
{
  "type": "ORDER_STATUS",
  "data": {
    "order_id": "order_xyz789",
    "status": "PICKED_UP",
    "partner": {
      "name": "John D.",
      "location": {...}
    },
    "eta_minutes": 10
  }
}
```

**2. Delivery Partner Location (Server → Customer):**

```json
{
  "type": "PARTNER_LOCATION",
  "data": {
    "order_id": "order_xyz789",
    "location": {
      "latitude": 37.7800,
      "longitude": -122.4150
    },
    "eta_minutes": 8
  }
}
```

**3. New Order (Server → Restaurant):**

```json
{
  "type": "NEW_ORDER",
  "data": {
    "order_id": "order_xyz789",
    "items_count": 3,
    "total": 43.91,
    "accept_deadline": "2024-01-15T18:35:00Z"
  }
}
```

**4. Location Update (Partner → Server):**

```json
{
  "type": "LOCATION_UPDATE",
  "data": {
    "latitude": 37.7800,
    "longitude": -122.4150,
    "heading": 45,
    "speed": 25
  }
}
```

---

## Database Schema Design

### Database Choice

| Database | Purpose | Why |
|----------|---------|-----|
| PostgreSQL | Transactional data | ACID for orders, payments |
| Elasticsearch | Restaurant search | Full-text, geo queries |
| Redis | Real-time data | Sessions, locations, cache |
| TimescaleDB | Location history | Time-series optimized |

### Core Tables

#### 1. restaurants Table

```sql
CREATE TABLE restaurants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    name VARCHAR(200) NOT NULL,
    description TEXT,
    
    address_line1 VARCHAR(255) NOT NULL,
    address_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state VARCHAR(50) NOT NULL,
    zip_code VARCHAR(20) NOT NULL,
    location GEOGRAPHY(POINT, 4326) NOT NULL,
    
    cuisine_types VARCHAR(50)[] DEFAULT '{}',
    price_level SMALLINT DEFAULT 2,
    
    rating DECIMAL(2, 1) DEFAULT 0,
    rating_count INT DEFAULT 0,
    
    delivery_radius_km DECIMAL(4, 1) DEFAULT 5.0,
    minimum_order DECIMAL(10, 2) DEFAULT 0,
    delivery_fee DECIMAL(10, 2) DEFAULT 0,
    avg_prep_time_minutes INT DEFAULT 20,
    
    operating_hours JSONB NOT NULL,
    is_active BOOLEAN DEFAULT true,
    is_accepting_orders BOOLEAN DEFAULT true,
    
    owner_id UUID REFERENCES users(id),
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_restaurants_location ON restaurants USING GIST(location);
CREATE INDEX idx_restaurants_cuisine ON restaurants USING GIN(cuisine_types);
CREATE INDEX idx_restaurants_active ON restaurants(is_active, is_accepting_orders);
```

#### 2. menu_items Table

```sql
CREATE TABLE menu_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    restaurant_id UUID NOT NULL REFERENCES restaurants(id),
    
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    
    category_id UUID REFERENCES menu_categories(id),
    
    photo_url VARCHAR(500),
    
    is_available BOOLEAN DEFAULT true,
    unavailable_until TIMESTAMP WITH TIME ZONE,
    
    prep_time_minutes INT DEFAULT 15,
    calories INT,
    dietary_tags VARCHAR(50)[] DEFAULT '{}',
    
    customizations JSONB DEFAULT '[]',
    
    display_order INT DEFAULT 0,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_menu_items_restaurant ON menu_items(restaurant_id, is_available);
CREATE INDEX idx_menu_items_category ON menu_items(category_id);
```

#### 3. orders Table

```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    customer_id UUID NOT NULL REFERENCES users(id),
    restaurant_id UUID NOT NULL REFERENCES restaurants(id),
    delivery_partner_id UUID REFERENCES users(id),
    
    status VARCHAR(20) NOT NULL DEFAULT 'PLACED',
    
    items JSONB NOT NULL,
    
    subtotal DECIMAL(10, 2) NOT NULL,
    discount DECIMAL(10, 2) DEFAULT 0,
    delivery_fee DECIMAL(10, 2) NOT NULL,
    service_fee DECIMAL(10, 2) NOT NULL,
    tax DECIMAL(10, 2) NOT NULL,
    tip DECIMAL(10, 2) DEFAULT 0,
    total DECIMAL(10, 2) NOT NULL,
    
    promo_code VARCHAR(50),
    promo_discount DECIMAL(10, 2) DEFAULT 0,
    
    delivery_address JSONB NOT NULL,
    delivery_instructions TEXT,
    
    special_instructions TEXT,
    
    scheduled_for TIMESTAMP WITH TIME ZONE,
    
    estimated_prep_time INT,
    estimated_delivery_time TIMESTAMP WITH TIME ZONE,
    
    placed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    confirmed_at TIMESTAMP WITH TIME ZONE,
    ready_at TIMESTAMP WITH TIME ZONE,
    picked_up_at TIMESTAMP WITH TIME ZONE,
    delivered_at TIMESTAMP WITH TIME ZONE,
    cancelled_at TIMESTAMP WITH TIME ZONE,
    
    cancellation_reason VARCHAR(50),
    cancelled_by VARCHAR(20),
    
    payment_method_id VARCHAR(100),
    payment_status VARCHAR(20) DEFAULT 'PENDING',
    
    customer_rating SMALLINT,
    restaurant_rating SMALLINT,
    partner_rating SMALLINT,
    
    CONSTRAINT valid_status CHECK (status IN (
        'PLACED', 'CONFIRMED', 'PREPARING', 'READY',
        'PICKED_UP', 'DELIVERED', 'CANCELLED'
    ))
);

CREATE INDEX idx_orders_customer ON orders(customer_id, placed_at DESC);
CREATE INDEX idx_orders_restaurant ON orders(restaurant_id, placed_at DESC);
CREATE INDEX idx_orders_partner ON orders(delivery_partner_id, placed_at DESC);
CREATE INDEX idx_orders_status ON orders(status) WHERE status NOT IN ('DELIVERED', 'CANCELLED');
```

#### 4. delivery_partners Table

```sql
CREATE TABLE delivery_partners (
    user_id UUID PRIMARY KEY REFERENCES users(id),
    
    vehicle_type VARCHAR(20) NOT NULL,
    vehicle_details VARCHAR(100),
    license_plate VARCHAR(20),
    
    status VARCHAR(20) DEFAULT 'OFFLINE',
    
    current_location GEOGRAPHY(POINT, 4326),
    location_updated_at TIMESTAMP WITH TIME ZONE,
    
    rating DECIMAL(2, 1) DEFAULT 5.0,
    total_deliveries INT DEFAULT 0,
    
    acceptance_rate DECIMAL(3, 2) DEFAULT 1.0,
    completion_rate DECIMAL(3, 2) DEFAULT 1.0,
    
    bank_account_id VARCHAR(100),
    
    background_check_status VARCHAR(20) DEFAULT 'PENDING',
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CONSTRAINT valid_status CHECK (status IN ('ONLINE', 'OFFLINE', 'BUSY'))
);

CREATE INDEX idx_partners_status ON delivery_partners(status) WHERE status = 'ONLINE';
CREATE INDEX idx_partners_location ON delivery_partners USING GIST(current_location);
```

#### 5. payments Table

```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    
    customer_id UUID NOT NULL REFERENCES users(id),
    
    amount DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    
    payment_method_id VARCHAR(100) NOT NULL,
    payment_provider VARCHAR(20) NOT NULL,
    provider_transaction_id VARCHAR(100),
    
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    
    restaurant_payout DECIMAL(10, 2),
    partner_payout DECIMAL(10, 2),
    platform_fee DECIMAL(10, 2),
    
    refund_amount DECIMAL(10, 2) DEFAULT 0,
    refund_reason VARCHAR(100),
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    processed_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_status CHECK (status IN (
        'PENDING', 'AUTHORIZED', 'CAPTURED', 'FAILED', 'REFUNDED', 'PARTIALLY_REFUNDED'
    ))
);

CREATE INDEX idx_payments_order ON payments(order_id);
CREATE INDEX idx_payments_customer ON payments(customer_id, created_at DESC);
```

---

## Entity Relationship Diagram

```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│     users       │       │  restaurants    │       │delivery_partners│
├─────────────────┤       ├─────────────────┤       ├─────────────────┤
│ id (PK)         │       │ id (PK)         │       │ user_id (PK,FK) │
│ name            │       │ name            │       │ vehicle_type    │
│ email           │       │ location        │       │ status          │
│ phone           │       │ cuisine_types   │       │ current_location│
│ type            │       │ rating          │       │ rating          │
└────────┬────────┘       └────────┬────────┘       └────────┬────────┘
         │                         │                         │
         │                         │                         │
         │                         ▼                         │
         │                ┌─────────────────┐                │
         │                │   menu_items    │                │
         │                ├─────────────────┤                │
         │                │ id (PK)         │                │
         │                │ restaurant_id   │                │
         │                │ name, price     │                │
         │                │ customizations  │                │
         │                └─────────────────┘                │
         │                                                   │
         │    ┌──────────────────────────────────────────────┘
         │    │
         ▼    ▼
┌─────────────────────────────────┐       ┌─────────────────┐
│            orders               │       │    payments     │
├─────────────────────────────────┤       ├─────────────────┤
│ id (PK)                         │◄──────│ order_id (FK)   │
│ customer_id (FK)                │       │ amount          │
│ restaurant_id (FK)              │       │ status          │
│ delivery_partner_id (FK)        │       │ payouts         │
│ status                          │       └─────────────────┘
│ items (JSONB)                   │
│ total, fees, tip                │
│ delivery_address                │
│ timestamps                      │
└─────────────────────────────────┘
```

---

## Delivery Assignment Algorithm

### Assignment Criteria

```java
public class DeliveryAssignmentService {
    
    public Optional<DeliveryPartner> findBestPartner(Order order) {
        Location restaurantLocation = order.getRestaurant().getLocation();
        
        // 1. Find nearby available partners
        List<DeliveryPartner> candidates = partnerRepository
            .findAvailableNear(restaurantLocation, 5.0);  // 5km radius
        
        if (candidates.isEmpty()) {
            return Optional.empty();
        }
        
        // 2. Score each partner
        List<ScoredPartner> scored = candidates.stream()
            .map(p -> scorePartner(p, order))
            .sorted(Comparator.comparing(ScoredPartner::getScore).reversed())
            .collect(Collectors.toList());
        
        return Optional.of(scored.get(0).getPartner());
    }
    
    private ScoredPartner scorePartner(DeliveryPartner partner, Order order) {
        double score = 0.0;
        
        // Distance to restaurant (40% weight)
        double distanceKm = calculateDistance(
            partner.getCurrentLocation(),
            order.getRestaurant().getLocation()
        );
        score += Math.max(0, 40 - (distanceKm * 8));  // 0-40 points
        
        // Rating (25% weight)
        score += (partner.getRating() - 4.0) * 25;  // 0-25 points
        
        // Acceptance rate (20% weight)
        score += partner.getAcceptanceRate() * 20;  // 0-20 points
        
        // Completion rate (15% weight)
        score += partner.getCompletionRate() * 15;  // 0-15 points
        
        return new ScoredPartner(partner, score);
    }
}
```

---

## Estimated Delivery Time Calculation

```java
public class ETAService {
    
    public DeliveryEstimate calculateETA(Order order) {
        // 1. Restaurant prep time
        int prepTime = order.getRestaurant().getAvgPrepTimeMinutes();
        
        // 2. Partner arrival at restaurant
        int partnerToRestaurant = 0;
        if (order.getDeliveryPartner() != null) {
            partnerToRestaurant = calculateTravelTime(
                order.getDeliveryPartner().getCurrentLocation(),
                order.getRestaurant().getLocation()
            );
        } else {
            partnerToRestaurant = 10;  // Estimated assignment + travel
        }
        
        // 3. Restaurant to customer
        int restaurantToCustomer = calculateTravelTime(
            order.getRestaurant().getLocation(),
            order.getDeliveryAddress().getLocation()
        );
        
        // 4. Buffer for variability
        int buffer = 5;
        
        int minMinutes = prepTime + restaurantToCustomer + buffer;
        int maxMinutes = prepTime + partnerToRestaurant + restaurantToCustomer + buffer + 10;
        
        return new DeliveryEstimate(minMinutes, maxMinutes);
    }
    
    private int calculateTravelTime(Location from, Location to) {
        double distanceKm = calculateDistance(from, to);
        // Assume average speed of 20 km/h in city
        return (int) Math.ceil(distanceKm / 20.0 * 60);
    }
}
```

---

## Idempotency Model

### What is Idempotency?

An operation is **idempotent** if performing it multiple times has the same effect as performing it once. This is critical for handling retries, network failures, and duplicate requests in a food delivery system.

### Idempotent Operations in Food Delivery

| Operation | Idempotent? | Mechanism |
|-----------|-------------|-----------|
| **Place Order** | ✅ Yes | Idempotency key prevents duplicate orders |
| **Cancel Order** | ✅ Yes | State machine prevents double cancellation |
| **Update Order Status** | ✅ Yes | Version-based updates prevent conflicts |
| **Process Payment** | ✅ Yes | Payment gateway idempotency keys |
| **Assign Delivery Partner** | ✅ Yes | Atomic assignment with lock |
| **Update Location** | ⚠️ At-least-once | Deduplication by (order_id, timestamp) |

### Idempotency Implementation

**1. Order Placement with Idempotency Key:**

```java
@PostMapping("/v1/orders")
public ResponseEntity<OrderResponse> createOrder(
        @RequestBody CreateOrderRequest request,
        @RequestHeader("Idempotency-Key") String idempotencyKey) {
    
    // Check if order already exists
    Order existing = orderRepository.findByIdempotencyKey(idempotencyKey);
    if (existing != null) {
        return ResponseEntity.ok(OrderResponse.from(existing));
    }
    
    // Validate request
    validateOrderRequest(request);
    
    // Create order
    Order order = orderService.createOrder(request, idempotencyKey);
    
    // Process payment (idempotent via payment gateway)
    PaymentResult payment = paymentService.authorize(
        order.getPaymentMethodId(),
        order.getTotalAmount(),
        idempotencyKey + "_payment"
    );
    
    if (!payment.isSuccess()) {
        throw new PaymentFailedException(payment.getError());
    }
    
    // Assign delivery partner (idempotent via lock)
    DeliveryAssignment assignment = deliveryService.assignPartner(
        order.getId(),
        order.getDeliveryAddress(),
        idempotencyKey + "_assignment"
    );
    
    return ResponseEntity.status(201).body(OrderResponse.from(order));
}
```

**2. Payment Idempotency:**

```java
public PaymentResult authorize(String paymentMethodId, BigDecimal amount, String idempotencyKey) {
    // Check if payment already processed
    Payment existing = paymentRepository.findByIdempotencyKey(idempotencyKey);
    if (existing != null) {
        return PaymentResult.from(existing);
    }
    
    // Call payment gateway with idempotency key
    PaymentGatewayResponse response = stripeClient.authorize(
        paymentMethodId,
        amount,
        idempotencyKey  // Stripe supports idempotency keys
    );
    
    // Store payment record
    Payment payment = Payment.builder()
        .orderId(orderId)
        .idempotencyKey(idempotencyKey)
        .status(response.getStatus())
        .transactionId(response.getTransactionId())
        .build();
    
    paymentRepository.save(payment);
    
    return PaymentResult.from(payment);
}
```

**3. Order Cancellation Idempotency:**

```java
@PostMapping("/v1/orders/{orderId}/cancel")
public ResponseEntity<OrderResponse> cancelOrder(
        @PathVariable String orderId,
        @RequestHeader("Idempotency-Key") String idempotencyKey) {
    
    Order order = orderRepository.findById(orderId);
    
    // Check if already cancelled
    if (order.getStatus() == OrderStatus.CANCELLED) {
        return ResponseEntity.ok(OrderResponse.from(order)); // Idempotent
    }
    
    // Validate cancellation is allowed
    if (!order.canCancel()) {
        throw new InvalidOrderStateException("Order cannot be cancelled");
    }
    
    // Cancel order (state machine ensures idempotency)
    order = orderService.cancelOrder(orderId, idempotencyKey);
    
    // Process refund (idempotent)
    if (order.getPaymentStatus() == PaymentStatus.CAPTURED) {
        refundService.processRefund(orderId, idempotencyKey + "_refund");
    }
    
    return ResponseEntity.ok(OrderResponse.from(order));
}
```

**4. Location Update Deduplication:**

```java
// Deduplicate location updates by (order_id, timestamp)
public void updateDeliveryLocation(String orderId, Location location) {
    long timestamp = System.currentTimeMillis() / 1000; // Round to second
    String dedupKey = String.format("location:%s:%d", orderId, timestamp);
    
    // Set if absent (idempotent)
    Boolean isNew = redisTemplate.opsForValue()
        .setIfAbsent(dedupKey, "1", Duration.ofMinutes(1));
    
    if (Boolean.TRUE.equals(isNew)) {
        // New location update, process it
        deliveryService.updateLocation(orderId, location);
        // Broadcast via WebSocket
        websocketService.broadcastLocationUpdate(orderId, location);
    }
    // Duplicate update, ignore
}
```

### Idempotency Key Generation

**Client-Side (Recommended):**
```javascript
// Generate UUID v4 for each request
const idempotencyKey = crypto.randomUUID();
fetch('/v1/orders', {
    headers: {
        'Idempotency-Key': idempotencyKey
    },
    body: orderData
});
```

**Server-Side (Fallback):**
```java
// If client doesn't provide key, generate from request hash
String idempotencyKey = DigestUtils.sha256Hex(
    request.getRestaurantId() + 
    request.getItems().toString() + 
    request.getDeliveryAddress().toString() +
    request.getCustomerId()
);
```

### Duplicate Detection Window

| Operation | Deduplication Window | Storage |
|-----------|---------------------|---------|
| Order Placement | 24 hours | PostgreSQL (idempotency_key unique index) |
| Payment Authorization | 24 hours | PostgreSQL + Payment Gateway |
| Order Cancellation | 24 hours | PostgreSQL (state machine) |
| Location Updates | 1 minute | Redis |
| Delivery Assignment | 1 hour | PostgreSQL (lock-based) |

### Idempotency Key Storage

```sql
-- Add idempotency_key column with unique index
ALTER TABLE orders ADD COLUMN idempotency_key VARCHAR(255);
CREATE UNIQUE INDEX idx_orders_idempotency_key ON orders(idempotency_key);

-- Add idempotency_key to payments
ALTER TABLE payments ADD COLUMN idempotency_key VARCHAR(255);
CREATE UNIQUE INDEX idx_payments_idempotency_key ON payments(idempotency_key);
```

---

## Error Model

### Standard Error Response

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message",
    "details": {},
    "request_id": "req_abc123"
  }
}
```

### Error Codes

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | INVALID_ADDRESS | Delivery address is invalid |
| 400 | ITEM_UNAVAILABLE | Menu item not available |
| 400 | BELOW_MINIMUM | Order below minimum amount |
| 400 | RESTAURANT_CLOSED | Restaurant not accepting orders |
| 401 | UNAUTHORIZED | Invalid token |
| 402 | PAYMENT_FAILED | Payment authorization failed |
| 404 | ORDER_NOT_FOUND | Order does not exist |
| 409 | ORDER_ALREADY_CANCELLED | Cannot modify cancelled order |
| 429 | RATE_LIMITED | Too many requests |
| 503 | NO_PARTNERS | No delivery partners available |

