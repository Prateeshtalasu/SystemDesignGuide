# Ticket Booking System - API & Schema Design

## API Design Philosophy

Ticket booking APIs prioritize:

1. **Real-time Availability**: Fast seat map updates
2. **Concurrency**: Handle simultaneous seat selections
3. **Idempotency**: Prevent duplicate bookings
4. **Performance**: Fast booking completion
5. **User Experience**: Clear error messages

---

## Base URL Structure

```
REST API:     https://api.bookmyshow.com/v1
WebSocket:    wss://ws.bookmyshow.com/v1
```

---

## API Versioning Strategy

We use URL path versioning (`/v1/`, `/v2/`) for backward compatibility.

---

## Rate Limiting Headers

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
```

**Rate Limits:**

| Endpoint Type | Requests/minute |
|---------------|-----------------|
| Seat Map Views | 300 |
| Seat Selection | 60 |
| Booking | 30 |
| Payment | 20 |

---

## Error Model

All error responses follow standard envelope structure:

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

**Error Codes:**

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | INVALID_INPUT | Request validation failed |
| 400 | SEATS_UNAVAILABLE | Selected seats not available |
| 400 | SEATS_ALREADY_LOCKED | Seats locked by another user |
| 409 | BOOKING_EXISTS | Booking already exists |
| 422 | PAYMENT_FAILED | Payment processing failed |
| 429 | RATE_LIMITED | Rate limit exceeded |

---

## Idempotency Implementation

**Idempotency-Key Header:**

```http
Idempotency-Key: <uuid-v4>
```

**Per-Endpoint Idempotency:**

| Endpoint | Idempotent? | Mechanism |
|----------|-------------|-----------|
| POST /v1/bookings | Yes | Idempotency-Key header |
| POST /v1/seats/lock | Yes | Idempotency-Key header |
| POST /v1/payments | Yes | Idempotency-Key header (critical) |

---

## Core API Endpoints

### 1. Seat Management

#### Get Seat Map

**Endpoint:** `GET /v1/shows/{show_id}/seats`

**Response (200 OK):**

```json
{
  "show_id": "show_123",
  "venue": {
    "venue_id": "venue_456",
    "name": "PVR Cinemas",
    "total_seats": 200
  },
  "seats": [
    {
      "seat_id": "seat_1",
      "row": "A",
      "number": 1,
      "status": "available",
      "price": 500,
      "zone": "premium"
    }
  ],
  "zones": {
    "premium": {"price": 500, "seats": 50},
    "standard": {"price": 300, "seats": 150}
  }
}
```

#### Lock Seats

**Endpoint:** `POST /v1/seats/lock`

**Request:**

```json
{
  "show_id": "show_123",
  "seat_ids": ["seat_1", "seat_2", "seat_3"],
  "lock_duration_seconds": 300
}
```

**Response (200 OK):**

```json
{
  "lock_id": "lock_abc123",
  "show_id": "show_123",
  "seat_ids": ["seat_1", "seat_2", "seat_3"],
  "expires_at": "2024-01-20T15:35:00Z",
  "locked_at": "2024-01-20T15:30:00Z"
}
```

**Error Response (400 Bad Request):**

```json
{
  "error": {
    "code": "SEATS_UNAVAILABLE",
    "message": "Some seats are no longer available",
    "details": {
      "unavailable_seats": ["seat_1", "seat_2"]
    }
  }
}
```

---

### 2. Booking Management

#### Create Booking

**Endpoint:** `POST /v1/bookings`

**Request:**

```json
{
  "show_id": "show_123",
  "lock_id": "lock_abc123",
  "seat_ids": ["seat_1", "seat_2", "seat_3"],
  "user_id": "user_789",
  "payment_method": {
    "type": "credit_card",
    "card_number": "4111111111111111",
    "expiry_month": 12,
    "expiry_year": 2025,
    "cvv": "123"
  }
}
```

**Response (201 Created):**

```json
{
  "booking": {
    "booking_id": "booking_123456",
    "show_id": "show_123",
    "user_id": "user_789",
    "seats": [
      {
        "seat_id": "seat_1",
        "row": "A",
        "number": 1,
        "price": 500
      }
    ],
    "total_amount": 1500,
    "status": "confirmed",
    "payment_id": "pay_abc123",
    "ticket_id": "ticket_xyz789",
    "created_at": "2024-01-20T15:30:00Z"
  }
}
```

#### Get Booking

**Endpoint:** `GET /v1/bookings/{booking_id}`

**Response (200 OK):**

```json
{
  "booking": {
    "booking_id": "booking_123456",
    "show": {
      "show_id": "show_123",
      "event_name": "Avengers: Endgame",
      "show_time": "2024-01-25T18:00:00Z",
      "venue": "PVR Cinemas"
    },
    "seats": [...],
    "total_amount": 1500,
    "status": "confirmed",
    "ticket_url": "https://tickets.bookmyshow.com/ticket_xyz789"
  }
}
```

---

### 3. Payment Processing

#### Process Payment

**Endpoint:** `POST /v1/payments`

**Request:**

```json
{
  "booking_id": "booking_123456",
  "amount": 1500,
  "payment_method": {
    "type": "credit_card",
    "card_number": "4111111111111111",
    "expiry_month": 12,
    "expiry_year": 2025,
    "cvv": "123"
  }
}
```

**Response (200 OK):**

```json
{
  "payment_id": "pay_abc123",
  "booking_id": "booking_123456",
  "status": "succeeded",
  "amount": 1500,
  "processed_at": "2024-01-20T15:30:00Z"
}
```

---

## Database Schema Design

### Bookings Schema

```sql
CREATE TABLE bookings (
    booking_id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL,
    show_id VARCHAR(50) NOT NULL,
    event_id VARCHAR(50) NOT NULL,
    
    status VARCHAR(20) DEFAULT 'pending',
    total_amount DECIMAL(10, 2) NOT NULL,
    
    payment_id VARCHAR(50),
    ticket_id VARCHAR(50),
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    confirmed_at TIMESTAMP WITH TIME ZONE,
    cancelled_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_status CHECK (status IN ('pending', 'confirmed', 'cancelled', 'expired'))
);

CREATE INDEX idx_bookings_user ON bookings(user_id, created_at DESC);
CREATE INDEX idx_bookings_show ON bookings(show_id);
CREATE INDEX idx_bookings_status ON bookings(status);
```

### Seat Locks Schema

```sql
CREATE TABLE seat_locks (
    lock_id VARCHAR(50) PRIMARY KEY,
    show_id VARCHAR(50) NOT NULL,
    seat_id VARCHAR(50) NOT NULL,
    user_id VARCHAR(50) NOT NULL,
    
    status VARCHAR(20) DEFAULT 'locked',
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    released_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_status CHECK (status IN ('locked', 'confirmed', 'released', 'expired')),
    UNIQUE(show_id, seat_id, status) WHERE status = 'locked'
);

CREATE INDEX idx_seat_locks_show_seat ON seat_locks(show_id, seat_id);
CREATE INDEX idx_seat_locks_expires ON seat_locks(expires_at) WHERE status = 'locked';
```

### Booking Seats Schema

```sql
CREATE TABLE booking_seats (
    id BIGSERIAL PRIMARY KEY,
    booking_id VARCHAR(50) NOT NULL,
    seat_id VARCHAR(50) NOT NULL,
    row VARCHAR(10) NOT NULL,
    number INTEGER NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    
    CONSTRAINT fk_booking FOREIGN KEY (booking_id) REFERENCES bookings(booking_id) ON DELETE CASCADE,
    UNIQUE(booking_id, seat_id)
);

CREATE INDEX idx_booking_seats_booking ON booking_seats(booking_id);
CREATE INDEX idx_booking_seats_seat ON booking_seats(seat_id);
```

---

## Summary

| Component | Technology/Approach |
|-----------|-------------------|
| API Versioning | URL path versioning (/v1/) |
| Authentication | JWT tokens |
| Idempotency | Idempotency-Key header + Redis |
| Seat Locking | Distributed locking + TTL |
| Booking Storage | PostgreSQL |
| Seat Availability | Redis (cache) + PostgreSQL (source of truth) |

