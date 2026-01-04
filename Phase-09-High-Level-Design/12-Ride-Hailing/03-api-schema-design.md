# Ride Hailing - API & Schema Design

## API Design Philosophy

Ride-hailing APIs must handle:
1. **Real-time communication**: WebSocket for location updates
2. **Idempotency**: Ride requests must not create duplicates
3. **State machines**: Rides transition through defined states
4. **Geographic data**: Locations as first-class citizens

---

## Base URL Structure

```
REST API:     https://api.ridehail.com/v1
WebSocket:    wss://ws.ridehail.com/v1
```

### API Versioning Strategy

We use URL path versioning (`/v1/`, `/v2/`) because:
- Clear in logs and documentation
- Easy to run multiple versions simultaneously
- Mobile apps can pin to specific versions

---

## Authentication

### JWT Token Authentication

```http
GET /v1/rides
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

### Token Structure

```json
{
  "sub": "user_123456",
  "type": "rider",  // or "driver"
  "exp": 1735689600,
  "iat": 1735603200,
  "device_id": "device_abc123"
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
| Authentication | 10 | 1 minute |
| Ride requests | 5 | 1 minute |
| Location updates | 1000 | 1 minute |
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
| 400 | NO_DRIVERS_AVAILABLE | No drivers available in area |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Ride or resource not found |
| 409 | CONFLICT | Ride state conflict |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

**Error Response Examples:**

```json
// 400 Bad Request - No drivers available
{
  "error": {
    "code": "NO_DRIVERS_AVAILABLE",
    "message": "No drivers available in your area",
    "details": {
      "field": "pickup",
      "reason": "No drivers within 5km radius"
    },
    "request_id": "req_abc123"
  }
}

// 409 Conflict - Ride state conflict
{
  "error": {
    "code": "CONFLICT",
    "message": "Ride state conflict",
    "details": {
      "field": "ride_id",
      "reason": "Ride is already in progress"
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
      "limit": 5,
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

Clients must include `Idempotency-Key` header for all write operations (critical for ride bookings):
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
| POST /v1/rides | Yes | Idempotency-Key header (CRITICAL - prevents duplicate bookings) |
| PUT /v1/rides/{id} | Yes | Idempotency-Key or version-based |
| POST /v1/rides/{id}/cancel | Yes | Idempotency-Key header |
| POST /v1/rides/{id}/rate | Yes | Idempotency-Key header |

**Implementation Example:**

```java
@PostMapping("/v1/rides")
public ResponseEntity<RideResponse> requestRide(
        @RequestBody RequestRideRequest request,
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {
    
    // Check for existing idempotency key (CRITICAL for preventing duplicate rides)
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        String cachedResponse = redisTemplate.opsForValue().get(cacheKey);
        if (cachedResponse != null) {
            RideResponse response = objectMapper.readValue(cachedResponse, RideResponse.class);
            return ResponseEntity.status(response.getStatus()).body(response);
        }
    }
    
    // Create ride request
    Ride ride = rideService.createRideRequest(request, idempotencyKey);
    RideResponse response = RideResponse.from(ride);
    
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

## Core API Endpoints

### 1. Rider: Request a Ride

**Endpoint:** `POST /v1/rides`

**Purpose:** Create a new ride request

**Request:**

```http
POST /v1/rides HTTP/1.1
Host: api.ridehail.com
Authorization: Bearer {token}
Content-Type: application/json
Idempotency-Key: ride_req_abc123

{
  "pickup": {
    "latitude": 37.7749,
    "longitude": -122.4194,
    "address": "123 Market St, San Francisco, CA"
  },
  "dropoff": {
    "latitude": 37.7849,
    "longitude": -122.4094,
    "address": "456 Mission St, San Francisco, CA"
  },
  "vehicle_type": "STANDARD",
  "payment_method_id": "pm_123456",
  "scheduled_at": null,
  "notes": "Meet at the lobby"
}
```

**Response (201 Created):**

```json
{
  "ride_id": "ride_789xyz",
  "status": "MATCHING",
  "pickup": {
    "latitude": 37.7749,
    "longitude": -122.4194,
    "address": "123 Market St, San Francisco, CA"
  },
  "dropoff": {
    "latitude": 37.7849,
    "longitude": -122.4094,
    "address": "456 Mission St, San Francisco, CA"
  },
  "vehicle_type": "STANDARD",
  "fare_estimate": {
    "min": 12.50,
    "max": 16.00,
    "currency": "USD",
    "surge_multiplier": 1.2
  },
  "eta_pickup": 4,
  "eta_dropoff": 12,
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Ride Status State Machine:**

```
MATCHING → DRIVER_ASSIGNED → DRIVER_EN_ROUTE → ARRIVED → IN_PROGRESS → COMPLETED
    ↓           ↓                  ↓              ↓           ↓
CANCELLED   CANCELLED         CANCELLED      CANCELLED   CANCELLED
```

**Error Responses:**

```json
// 400 Bad Request - Invalid location
{
  "error": {
    "code": "INVALID_LOCATION",
    "message": "Pickup location is outside service area",
    "details": {
      "field": "pickup",
      "service_area": "San Francisco Bay Area"
    }
  }
}

// 402 Payment Required - Payment method failed
{
  "error": {
    "code": "PAYMENT_FAILED",
    "message": "Unable to authorize payment method",
    "details": {
      "reason": "Card declined"
    }
  }
}

// 503 Service Unavailable - No drivers available
{
  "error": {
    "code": "NO_DRIVERS",
    "message": "No drivers available in your area. Try again shortly.",
    "retry_after": 60
  }
}
```

---

### 2. Get Fare Estimate

**Endpoint:** `GET /v1/rides/estimate`

**Purpose:** Get fare estimate before booking

**Request:**

```http
GET /v1/rides/estimate?pickup_lat=37.7749&pickup_lng=-122.4194&dropoff_lat=37.7849&dropoff_lng=-122.4094&vehicle_type=STANDARD HTTP/1.1
Host: api.ridehail.com
Authorization: Bearer {token}
```

**Response (200 OK):**

```json
{
  "estimates": [
    {
      "vehicle_type": "STANDARD",
      "fare": {
        "min": 12.50,
        "max": 16.00,
        "currency": "USD"
      },
      "surge_multiplier": 1.2,
      "eta_pickup": 4,
      "eta_trip": 12,
      "distance_km": 3.2,
      "available_drivers": 15
    },
    {
      "vehicle_type": "PREMIUM",
      "fare": {
        "min": 22.00,
        "max": 28.00,
        "currency": "USD"
      },
      "surge_multiplier": 1.0,
      "eta_pickup": 7,
      "eta_trip": 12,
      "distance_km": 3.2,
      "available_drivers": 5
    },
    {
      "vehicle_type": "XL",
      "fare": {
        "min": 18.00,
        "max": 24.00,
        "currency": "USD"
      },
      "surge_multiplier": 1.0,
      "eta_pickup": 10,
      "eta_trip": 12,
      "distance_km": 3.2,
      "available_drivers": 3
    }
  ],
  "surge_info": {
    "is_surging": true,
    "reason": "High demand in this area",
    "expires_at": "2024-01-15T10:45:00Z"
  }
}
```

---

### 3. Get Ride Status

**Endpoint:** `GET /v1/rides/{ride_id}`

**Purpose:** Get current status of a ride

**Response (200 OK):**

```json
{
  "ride_id": "ride_789xyz",
  "status": "DRIVER_EN_ROUTE",
  "rider": {
    "id": "user_123",
    "name": "John D.",
    "phone_masked": "***-***-4567",
    "rating": 4.8
  },
  "driver": {
    "id": "driver_456",
    "name": "Sarah M.",
    "phone_masked": "***-***-8901",
    "rating": 4.9,
    "photo_url": "https://cdn.ridehail.com/photos/driver_456.jpg",
    "vehicle": {
      "make": "Toyota",
      "model": "Camry",
      "color": "Silver",
      "license_plate": "ABC 1234"
    },
    "location": {
      "latitude": 37.7755,
      "longitude": -122.4180,
      "heading": 45,
      "updated_at": "2024-01-15T10:32:15Z"
    }
  },
  "pickup": {
    "latitude": 37.7749,
    "longitude": -122.4194,
    "address": "123 Market St, San Francisco, CA"
  },
  "dropoff": {
    "latitude": 37.7849,
    "longitude": -122.4094,
    "address": "456 Mission St, San Francisco, CA"
  },
  "eta_pickup": 2,
  "fare_estimate": {
    "min": 12.50,
    "max": 16.00,
    "currency": "USD"
  },
  "created_at": "2024-01-15T10:30:00Z",
  "driver_assigned_at": "2024-01-15T10:30:30Z"
}
```

---

### 4. Cancel Ride

**Endpoint:** `POST /v1/rides/{ride_id}/cancel`

**Request:**

```http
POST /v1/rides/ride_789xyz/cancel HTTP/1.1
Host: api.ridehail.com
Authorization: Bearer {token}
Content-Type: application/json

{
  "reason": "CHANGED_PLANS",
  "comment": "No longer need the ride"
}
```

**Response (200 OK):**

```json
{
  "ride_id": "ride_789xyz",
  "status": "CANCELLED",
  "cancellation_fee": 0.00,
  "cancelled_by": "RIDER",
  "cancelled_at": "2024-01-15T10:31:00Z"
}
```

**Cancellation Fee Rules:**

| Timing | Fee |
|--------|-----|
| Before driver assigned | $0 |
| Within 2 min of assignment | $0 |
| After 2 min, before arrival | $5 |
| After driver arrived | $10 |

---

### 5. Driver: Go Online/Offline

**Endpoint:** `PUT /v1/drivers/me/status`

**Request:**

```http
PUT /v1/drivers/me/status HTTP/1.1
Host: api.ridehail.com
Authorization: Bearer {driver_token}
Content-Type: application/json

{
  "status": "ONLINE",
  "location": {
    "latitude": 37.7749,
    "longitude": -122.4194
  },
  "vehicle_types": ["STANDARD", "XL"]
}
```

**Response (200 OK):**

```json
{
  "driver_id": "driver_456",
  "status": "ONLINE",
  "online_since": "2024-01-15T10:00:00Z",
  "location": {
    "latitude": 37.7749,
    "longitude": -122.4194
  },
  "vehicle_types": ["STANDARD", "XL"],
  "acceptance_rate": 0.92,
  "today_earnings": 145.50
}
```

---

### 6. Driver: Accept/Decline Ride

**Endpoint:** `POST /v1/rides/{ride_id}/respond`

**Request:**

```http
POST /v1/rides/ride_789xyz/respond HTTP/1.1
Host: api.ridehail.com
Authorization: Bearer {driver_token}
Content-Type: application/json

{
  "action": "ACCEPT"
}
```

**Response (200 OK):**

```json
{
  "ride_id": "ride_789xyz",
  "status": "DRIVER_EN_ROUTE",
  "pickup": {
    "latitude": 37.7749,
    "longitude": -122.4194,
    "address": "123 Market St, San Francisco, CA"
  },
  "rider": {
    "name": "John D.",
    "rating": 4.8
  },
  "navigation_url": "https://maps.google.com/...",
  "eta_pickup": 4
}
```

---

### 7. Driver: Update Trip Status

**Endpoint:** `POST /v1/rides/{ride_id}/status`

**Request:**

```http
POST /v1/rides/ride_789xyz/status HTTP/1.1
Host: api.ridehail.com
Authorization: Bearer {driver_token}
Content-Type: application/json

{
  "status": "ARRIVED",
  "location": {
    "latitude": 37.7749,
    "longitude": -122.4194
  }
}
```

**Valid Status Transitions (Driver):**

| Current Status | Allowed Next Status |
|----------------|---------------------|
| DRIVER_EN_ROUTE | ARRIVED, CANCELLED |
| ARRIVED | IN_PROGRESS, CANCELLED |
| IN_PROGRESS | COMPLETED |

---

### 8. Rate Trip

**Endpoint:** `POST /v1/rides/{ride_id}/rating`

**Request (Rider rating driver):**

```http
POST /v1/rides/ride_789xyz/rating HTTP/1.1
Host: api.ridehail.com
Authorization: Bearer {token}
Content-Type: application/json

{
  "rating": 5,
  "comment": "Great driver, very professional",
  "badges": ["EXCELLENT_SERVICE", "CLEAN_CAR"],
  "tip": 3.00
}
```

**Response (201 Created):**

```json
{
  "rating_id": "rating_abc123",
  "ride_id": "ride_789xyz",
  "rating": 5,
  "tip_amount": 3.00,
  "created_at": "2024-01-15T10:50:00Z"
}
```

---

## WebSocket API (Real-Time Updates)

### Connection

```javascript
const ws = new WebSocket('wss://ws.ridehail.com/v1?token={jwt_token}');
```

### Message Types

**1. Location Update (Driver → Server):**

```json
{
  "type": "LOCATION_UPDATE",
  "data": {
    "latitude": 37.7755,
    "longitude": -122.4180,
    "heading": 45,
    "speed": 25,
    "accuracy": 10,
    "timestamp": "2024-01-15T10:32:15Z"
  }
}
```

**2. Ride Request (Server → Driver):**

```json
{
  "type": "RIDE_REQUEST",
  "data": {
    "ride_id": "ride_789xyz",
    "pickup": {
      "latitude": 37.7749,
      "longitude": -122.4194,
      "address": "123 Market St"
    },
    "dropoff": {
      "latitude": 37.7849,
      "longitude": -122.4094,
      "address": "456 Mission St"
    },
    "rider": {
      "name": "John D.",
      "rating": 4.8
    },
    "fare_estimate": 14.00,
    "distance_km": 3.2,
    "expires_at": "2024-01-15T10:30:15Z"
  }
}
```

**3. Ride Status Update (Server → Rider/Driver):**

```json
{
  "type": "RIDE_STATUS",
  "data": {
    "ride_id": "ride_789xyz",
    "status": "DRIVER_EN_ROUTE",
    "driver_location": {
      "latitude": 37.7755,
      "longitude": -122.4180
    },
    "eta_pickup": 3
  }
}
```

**4. Driver Location Update (Server → Rider):**

```json
{
  "type": "DRIVER_LOCATION",
  "data": {
    "ride_id": "ride_789xyz",
    "location": {
      "latitude": 37.7755,
      "longitude": -122.4180,
      "heading": 45
    },
    "eta_pickup": 2
  }
}
```

---

## Database Schema Design

### Database Choice: PostgreSQL + Redis + TimescaleDB

| Database | Purpose | Why |
|----------|---------|-----|
| PostgreSQL | Users, rides, payments | ACID, complex queries |
| Redis | Driver locations, sessions | Sub-ms geospatial queries |
| TimescaleDB | Location history | Time-series optimized |

### Core Tables

#### 1. users Table

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number VARCHAR(15) UNIQUE NOT NULL,
    email VARCHAR(255),
    name VARCHAR(100) NOT NULL,
    profile_photo_url VARCHAR(500),
    
    user_type VARCHAR(10) NOT NULL,  -- 'RIDER' or 'DRIVER'
    
    rating DECIMAL(3, 2) DEFAULT 5.00,
    total_ratings INT DEFAULT 0,
    total_rides INT DEFAULT 0,
    
    status VARCHAR(20) DEFAULT 'ACTIVE',
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CONSTRAINT valid_user_type CHECK (user_type IN ('RIDER', 'DRIVER')),
    CONSTRAINT valid_status CHECK (status IN ('ACTIVE', 'SUSPENDED', 'DELETED'))
);

CREATE INDEX idx_users_phone ON users(phone_number);
CREATE INDEX idx_users_type_status ON users(user_type, status);
```

#### 2. drivers Table (Additional Driver Info)

```sql
CREATE TABLE drivers (
    user_id UUID PRIMARY KEY REFERENCES users(id),
    
    license_number VARCHAR(50) NOT NULL,
    license_expiry DATE NOT NULL,
    
    vehicle_make VARCHAR(50) NOT NULL,
    vehicle_model VARCHAR(50) NOT NULL,
    vehicle_year INT NOT NULL,
    vehicle_color VARCHAR(30) NOT NULL,
    license_plate VARCHAR(20) NOT NULL,
    vehicle_types VARCHAR(50)[] DEFAULT ARRAY['STANDARD'],
    
    insurance_number VARCHAR(50),
    insurance_expiry DATE,
    
    bank_account_id VARCHAR(100),
    
    background_check_status VARCHAR(20) DEFAULT 'PENDING',
    background_check_date DATE,
    
    online_status VARCHAR(20) DEFAULT 'OFFLINE',
    last_location GEOGRAPHY(POINT, 4326),
    last_location_at TIMESTAMP WITH TIME ZONE,
    
    acceptance_rate DECIMAL(3, 2) DEFAULT 1.00,
    cancellation_rate DECIMAL(3, 2) DEFAULT 0.00,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CONSTRAINT valid_online_status CHECK (online_status IN ('ONLINE', 'OFFLINE', 'BUSY'))
);

CREATE INDEX idx_drivers_online ON drivers(online_status) WHERE online_status = 'ONLINE';
CREATE INDEX idx_drivers_location ON drivers USING GIST(last_location);
```

#### 3. rides Table

```sql
CREATE TABLE rides (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    rider_id UUID NOT NULL REFERENCES users(id),
    driver_id UUID REFERENCES users(id),
    
    status VARCHAR(20) NOT NULL DEFAULT 'MATCHING',
    
    pickup_location GEOGRAPHY(POINT, 4326) NOT NULL,
    pickup_address VARCHAR(500) NOT NULL,
    dropoff_location GEOGRAPHY(POINT, 4326) NOT NULL,
    dropoff_address VARCHAR(500) NOT NULL,
    
    vehicle_type VARCHAR(20) NOT NULL,
    
    fare_estimate_min DECIMAL(10, 2),
    fare_estimate_max DECIMAL(10, 2),
    final_fare DECIMAL(10, 2),
    surge_multiplier DECIMAL(3, 2) DEFAULT 1.00,
    
    distance_km DECIMAL(10, 2),
    duration_minutes INT,
    
    scheduled_at TIMESTAMP WITH TIME ZONE,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    matched_at TIMESTAMP WITH TIME ZONE,
    driver_arrived_at TIMESTAMP WITH TIME ZONE,
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    cancelled_at TIMESTAMP WITH TIME ZONE,
    
    cancelled_by VARCHAR(10),
    cancellation_reason VARCHAR(50),
    cancellation_fee DECIMAL(10, 2) DEFAULT 0,
    
    rider_rating SMALLINT,
    driver_rating SMALLINT,
    
    payment_method_id VARCHAR(100),
    payment_status VARCHAR(20) DEFAULT 'PENDING',
    
    route_polyline TEXT,
    
    CONSTRAINT valid_status CHECK (status IN (
        'MATCHING', 'DRIVER_ASSIGNED', 'DRIVER_EN_ROUTE', 
        'ARRIVED', 'IN_PROGRESS', 'COMPLETED', 'CANCELLED'
    )),
    CONSTRAINT valid_vehicle_type CHECK (vehicle_type IN ('STANDARD', 'PREMIUM', 'XL', 'POOL')),
    CONSTRAINT valid_rating CHECK (rider_rating BETWEEN 1 AND 5 AND driver_rating BETWEEN 1 AND 5)
);

CREATE INDEX idx_rides_rider ON rides(rider_id, created_at DESC);
CREATE INDEX idx_rides_driver ON rides(driver_id, created_at DESC);
CREATE INDEX idx_rides_status ON rides(status) WHERE status NOT IN ('COMPLETED', 'CANCELLED');
CREATE INDEX idx_rides_created ON rides(created_at DESC);
```

#### 4. ride_events Table (State Changes)

```sql
CREATE TABLE ride_events (
    id BIGSERIAL PRIMARY KEY,
    ride_id UUID NOT NULL REFERENCES rides(id),
    
    event_type VARCHAR(30) NOT NULL,
    previous_status VARCHAR(20),
    new_status VARCHAR(20),
    
    actor_id UUID,
    actor_type VARCHAR(10),
    
    location GEOGRAPHY(POINT, 4326),
    metadata JSONB DEFAULT '{}',
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_ride_events_ride ON ride_events(ride_id, created_at);
```

#### 5. payments Table

```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ride_id UUID NOT NULL REFERENCES rides(id),
    
    rider_id UUID NOT NULL REFERENCES users(id),
    driver_id UUID NOT NULL REFERENCES users(id),
    
    amount DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    
    platform_fee DECIMAL(10, 2) NOT NULL,
    driver_payout DECIMAL(10, 2) NOT NULL,
    tip_amount DECIMAL(10, 2) DEFAULT 0,
    
    payment_method_id VARCHAR(100) NOT NULL,
    payment_provider VARCHAR(20) NOT NULL,
    provider_transaction_id VARCHAR(100),
    
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    processed_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_payment_status CHECK (status IN ('PENDING', 'AUTHORIZED', 'CAPTURED', 'FAILED', 'REFUNDED'))
);

CREATE INDEX idx_payments_ride ON payments(ride_id);
CREATE INDEX idx_payments_driver ON payments(driver_id, created_at DESC);
```

#### 6. surge_zones Table

```sql
CREATE TABLE surge_zones (
    id SERIAL PRIMARY KEY,
    zone_id VARCHAR(50) NOT NULL,
    
    boundary GEOGRAPHY(POLYGON, 4326) NOT NULL,
    
    surge_multiplier DECIMAL(3, 2) DEFAULT 1.00,
    demand_level VARCHAR(10) DEFAULT 'NORMAL',
    
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_demand CHECK (demand_level IN ('LOW', 'NORMAL', 'HIGH', 'VERY_HIGH'))
);

CREATE INDEX idx_surge_zones_boundary ON surge_zones USING GIST(boundary);
CREATE INDEX idx_surge_zones_active ON surge_zones(expires_at) WHERE expires_at > NOW();
```

### Redis Data Structures

#### Driver Locations (Geospatial)

```redis
# Add driver location
GEOADD drivers:locations {longitude} {latitude} {driver_id}

# Find drivers within 5km
GEORADIUS drivers:locations {longitude} {latitude} 5 km WITHCOORD WITHDIST COUNT 20

# Driver details hash
HSET driver:{driver_id} status "ONLINE" vehicle_type "STANDARD,XL" rating "4.9"
```

#### Active Rides

```redis
# Ride state
HSET ride:{ride_id} status "DRIVER_EN_ROUTE" driver_id "driver_456" rider_id "user_123"

# Driver's current ride
SET driver:{driver_id}:current_ride {ride_id}

# Rider's current ride
SET rider:{rider_id}:current_ride {ride_id}
```

### TimescaleDB (Location History)

```sql
CREATE TABLE location_history (
    time TIMESTAMPTZ NOT NULL,
    ride_id UUID NOT NULL,
    user_id UUID NOT NULL,
    user_type VARCHAR(10) NOT NULL,
    
    latitude DOUBLE PRECISION NOT NULL,
    longitude DOUBLE PRECISION NOT NULL,
    speed REAL,
    heading REAL,
    accuracy REAL
);

SELECT create_hypertable('location_history', 'time');

CREATE INDEX idx_location_ride ON location_history(ride_id, time DESC);
```

---

## Entity Relationship Diagram

```
┌─────────────────┐       ┌─────────────────┐
│     users       │       │    drivers      │
├─────────────────┤       ├─────────────────┤
│ id (PK)         │◄──────│ user_id (PK,FK) │
│ phone_number    │       │ license_number  │
│ name            │       │ vehicle_*       │
│ user_type       │       │ online_status   │
│ rating          │       │ last_location   │
└────────┬────────┘       └─────────────────┘
         │
         │ 1:N (as rider)
         │ 1:N (as driver)
         ▼
┌─────────────────┐       ┌─────────────────┐
│     rides       │       │   ride_events   │
├─────────────────┤       ├─────────────────┤
│ id (PK)         │◄──────│ ride_id (FK)    │
│ rider_id (FK)   │       │ event_type      │
│ driver_id (FK)  │       │ status_change   │
│ status          │       │ created_at      │
│ pickup_*        │       └─────────────────┘
│ dropoff_*       │
│ fare_*          │
└────────┬────────┘
         │
         │ 1:1
         ▼
┌─────────────────┐
│    payments     │
├─────────────────┤
│ id (PK)         │
│ ride_id (FK)    │
│ amount          │
│ driver_payout   │
│ status          │
└─────────────────┘
```

---

## Fare Calculation

### Pricing Components

```java
public class FareCalculator {
    
    private static final BigDecimal BASE_FARE = new BigDecimal("2.50");
    private static final BigDecimal PER_KM_RATE = new BigDecimal("1.50");
    private static final BigDecimal PER_MINUTE_RATE = new BigDecimal("0.25");
    private static final BigDecimal MINIMUM_FARE = new BigDecimal("5.00");
    private static final BigDecimal BOOKING_FEE = new BigDecimal("2.00");
    
    public FareEstimate calculate(
            double distanceKm, 
            int durationMinutes,
            VehicleType vehicleType,
            BigDecimal surgeMultiplier) {
        
        BigDecimal vehicleMultiplier = getVehicleMultiplier(vehicleType);
        
        BigDecimal distanceFare = PER_KM_RATE
            .multiply(BigDecimal.valueOf(distanceKm))
            .multiply(vehicleMultiplier);
            
        BigDecimal timeFare = PER_MINUTE_RATE
            .multiply(BigDecimal.valueOf(durationMinutes))
            .multiply(vehicleMultiplier);
            
        BigDecimal subtotal = BASE_FARE
            .add(distanceFare)
            .add(timeFare)
            .multiply(surgeMultiplier);
            
        BigDecimal total = subtotal.add(BOOKING_FEE);
        
        // Apply minimum fare
        if (total.compareTo(MINIMUM_FARE) < 0) {
            total = MINIMUM_FARE;
        }
        
        return FareEstimate.builder()
            .baseFare(BASE_FARE)
            .distanceFare(distanceFare)
            .timeFare(timeFare)
            .surgeMultiplier(surgeMultiplier)
            .bookingFee(BOOKING_FEE)
            .total(total)
            .build();
    }
    
    private BigDecimal getVehicleMultiplier(VehicleType type) {
        return switch (type) {
            case STANDARD -> BigDecimal.ONE;
            case PREMIUM -> new BigDecimal("1.75");
            case XL -> new BigDecimal("1.50");
            case POOL -> new BigDecimal("0.70");
        };
    }
}
```

---

## Idempotency Strategy

### Ride Request Idempotency

```java
@PostMapping("/v1/rides")
public ResponseEntity<RideResponse> createRide(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody RideRequest request) {
    
    // Check if we've seen this request before
    Optional<Ride> existingRide = rideRepository
        .findByIdempotencyKey(idempotencyKey);
    
    if (existingRide.isPresent()) {
        return ResponseEntity.ok(toResponse(existingRide.get()));
    }
    
    // Create new ride with idempotency key
    Ride ride = rideService.createRide(request, idempotencyKey);
    return ResponseEntity.status(201).body(toResponse(ride));
}
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
    "retry_after": 60,
    "request_id": "req_abc123"
  }
}
```

### Error Codes

| HTTP Status | Error Code | Description | Retryable |
|-------------|------------|-------------|-----------|
| 400 | INVALID_LOCATION | Location outside service area | No |
| 400 | INVALID_VEHICLE_TYPE | Vehicle type not available | No |
| 401 | UNAUTHORIZED | Invalid or expired token | No |
| 402 | PAYMENT_FAILED | Payment authorization failed | No |
| 403 | ACCOUNT_SUSPENDED | User account is suspended | No |
| 404 | RIDE_NOT_FOUND | Ride does not exist | No |
| 409 | RIDE_ALREADY_EXISTS | Duplicate ride request | No |
| 409 | DRIVER_BUSY | Driver already has a ride | No |
| 429 | RATE_LIMITED | Too many requests | Yes |
| 503 | NO_DRIVERS | No drivers available | Yes |
| 503 | SERVICE_UNAVAILABLE | System temporarily unavailable | Yes |

---

## Backward Compatibility

### API Versioning Rules

1. **Non-breaking changes** (no version bump):
   - Adding new optional fields
   - Adding new endpoints
   - Adding new enum values

2. **Breaking changes** (require new version):
   - Removing fields
   - Changing field types
   - Changing endpoint paths
   - Changing authentication

### Mobile App Compatibility

- Support last 2 major app versions
- Force update for security issues
- Graceful degradation for missing features

