# Centralized Logging System - API & Schema Design

## API Design Philosophy

The logging system has two types of APIs:

1. **Log Ingestion API**: High-throughput endpoint for services to send logs (HTTP, syslog, file-based)
2. **Query API**: RESTful API for searching and retrieving logs (for UI and integrations)

---

## Base URL Structure

```
Log Ingestion:  https://collector.logging.internal:8080 (HTTP)
                syslog://collector.logging.internal:514 (Syslog UDP)
                file:///var/log (File-based)
Query API:      https://api.logging.internal/v1
Dashboard:      https://logging.internal
```

---

## Log Ingestion API

### HTTP API (Push Model)

**Endpoint:** `POST /v1/logs`

**Request:**

```http
POST /v1/logs HTTP/1.1
Host: collector.logging.internal
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "logs": [
    {
      "timestamp": "2024-01-15T10:30:45.123Z",
      "level": "ERROR",
      "service": "order-service",
      "message": "Payment processing failed",
      "fields": {
        "order_id": "ORD-123",
        "user_id": "user_456",
        "error": "Payment gateway timeout"
      },
      "trace_id": "abc123def456"
    }
  ]
}
```

**Response:** `202 Accepted`

```json
{
  "status": "accepted",
  "logs_received": 1,
  "timestamp": 1705312245100
}
```

### Syslog Protocol (UDP)

**Endpoint:** UDP `collector.logging.internal:514`

**Format (RFC 3164):**

```
<134>Jan 15 10:30:45 order-service Payment processing failed order_id=ORD-123
```

---

## Query API

### 1. Search Logs

**Endpoint:** `GET /v1/logs/search`

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| query | string | Search query (full-text or field-based) |
| service | string | Filter by service name |
| level | string | Filter by log level (DEBUG, INFO, WARN, ERROR) |
| start_time | integer | Start timestamp (Unix timestamp) |
| end_time | integer | End timestamp (Unix timestamp) |
| limit | integer | Maximum number of logs (default: 100, max: 10,000) |
| offset | integer | Pagination offset |

**Request:**

```http
GET /v1/logs/search?query=Payment+processing+failed&service=order-service&level=ERROR&start_time=1705312200&end_time=1705312260&limit=100 HTTP/1.1
Host: api.logging.internal
Authorization: Bearer <token>
```

**Response:** `200 OK`

```json
{
  "logs": [
    {
      "timestamp": "2024-01-15T10:30:45.123Z",
      "level": "ERROR",
      "service": "order-service",
      "message": "Payment processing failed",
      "fields": {
        "order_id": "ORD-123",
        "user_id": "user_456",
        "error": "Payment gateway timeout"
      },
      "trace_id": "abc123def456"
    }
  ],
  "total": 42,
  "limit": 100,
  "offset": 0
}
```

### 2. Stream Logs (Real-time)

**Endpoint:** `GET /v1/logs/stream`

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| service | string | Filter by service name |
| level | string | Filter by log level |
| follow | boolean | Stream logs continuously (like tail -f) |

**Request:**

```http
GET /v1/logs/stream?service=order-service&level=ERROR&follow=true HTTP/1.1
Host: api.logging.internal
Authorization: Bearer <token>
```

**Response:** `200 OK` (Server-Sent Events)

```
data: {"timestamp":"2024-01-15T10:30:45.123Z","level":"ERROR","service":"order-service","message":"Payment processing failed"}

data: {"timestamp":"2024-01-15T10:30:46.456Z","level":"ERROR","service":"order-service","message":"Payment processing failed"}
```

### 3. Get Log by ID

**Endpoint:** `GET /v1/logs/{log_id}`

**Purpose:** Retrieve a specific log entry by ID

**Response:** `200 OK`

```json
{
  "id": "log_abc123",
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "ERROR",
  "service": "order-service",
  "message": "Payment processing failed",
  "fields": {
    "order_id": "ORD-123",
    "user_id": "user_456"
  },
  "trace_id": "abc123def456"
}
```

### 4. Aggregate Logs

**Endpoint:** `GET /v1/logs/aggregate`

**Purpose:** Aggregate logs by service, level, time range

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| group_by | string | Group by field (service, level, etc.) |
| start_time | integer | Start timestamp |
| end_time | integer | End timestamp |
| filter | string | Filter query |

**Response:** `200 OK`

```json
{
  "aggregations": [
    {
      "service": "order-service",
      "level": "ERROR",
      "count": 150
    },
    {
      "service": "payment-service",
      "level": "ERROR",
      "count": 25
    }
  ]
}
```

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
| 400 | INVALID_QUERY | Search query is invalid |
| 400 | INVALID_TIME_RANGE | Time range is invalid |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | LOG_NOT_FOUND | Log not found |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

---

## Idempotency Implementation

**Idempotency-Key Header:**

For log ingestion endpoints, clients can include `Idempotency-Key` header to prevent duplicate log processing:

```http
Idempotency-Key: <uuid-v4>
```

**Deduplication Storage:**

Idempotency keys are stored in Redis with 24-hour TTL:
```
Key: idempotency:{key}
Value: Serialized response (log ingestion confirmation)
TTL: 24 hours
```

**Retry Semantics:**

1. Client sends log batch with Idempotency-Key
2. Server checks Redis for existing key
3. If found: Return cached response (same status code + body)
4. If not found: Process logs, cache response, return result
5. Retries with same key within 24 hours return cached response

**Per-Endpoint Idempotency:**

| Endpoint | Idempotent? | Mechanism |
|----------|-------------|-----------|
| POST /v1/logs | ✅ Yes | Idempotency-Key header (optional, recommended for critical logs) |
| GET /v1/logs/search | ✅ Yes | Read-only, no side effects |
| GET /v1/logs/stream | ✅ Yes | Read-only, no side effects |
| GET /v1/logs/{log_id} | ✅ Yes | Read-only, no side effects |

**Implementation Example:**

```java
@PostMapping("/v1/logs")
public ResponseEntity<LogIngestionResponse> ingestLogs(
        @RequestBody LogIngestionRequest request,
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {
    
    // Check if request already processed (if idempotency key provided)
    if (idempotencyKey != null) {
        String cachedResponse = redisTemplate.opsForValue()
            .get("idempotency:" + idempotencyKey);
        if (cachedResponse != null) {
            return ResponseEntity.ok(parseResponse(cachedResponse));
        }
    }
    
    // Process logs
    LogIngestionResponse response = logService.ingestLogs(request);
    
    // Cache response for 24 hours (if idempotency key provided)
    if (idempotencyKey != null) {
        redisTemplate.opsForValue().set(
            "idempotency:" + idempotencyKey,
            serialize(response),
            Duration.ofHours(24)
        );
    }
    
    return ResponseEntity.status(202).body(response);
}
```

**Note:** Idempotency is optional for log ingestion (logs are append-only), but recommended for critical logs to prevent duplicate processing in case of retries.

---

## Rate Limiting Headers

Every response includes rate limit information:

```http
X-RateLimit-Limit: 10000
X-RateLimit-Remaining: 9999
X-RateLimit-Reset: 1640000000
```

**Rate Limits:**

| Endpoint Type | Requests/minute |
|---------------|-----------------|
| Log Ingestion | 1,000,000 per service |
| Search Query | 1,000 |
| Stream | 100 concurrent streams |

---

## Authentication/Authorization

**Log Ingestion:**
- API key authentication (per service)
- Keys stored in secure vault

**Query API:**
- OAuth 2.0 / JWT Bearer token
- RBAC: Read-only, Admin roles

---

## Summary

| Aspect | Decision |
|--------|----------|
| Log Ingestion | HTTP (push), Syslog (UDP), File-based |
| Query API | RESTful HTTP API with search capabilities |
| Authentication | API keys (ingestion), OAuth 2.0/JWT (query) |
| Rate Limiting | Per-service for ingestion, per-user for queries |
| Error Model | Standard error envelope with codes and details |

