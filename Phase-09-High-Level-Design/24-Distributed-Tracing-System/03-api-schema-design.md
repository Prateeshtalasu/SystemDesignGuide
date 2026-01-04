# Distributed Tracing System - API & Schema Design

## API Design Philosophy

The tracing system has two types of APIs:

1. **Span Ingestion API**: High-throughput, low-latency endpoint for services to send spans (OpenTelemetry, Jaeger, Zipkin protocols)
2. **Query API**: RESTful API for searching and retrieving traces (for UI and integrations)

---

## Base URL Structure

```
Span Ingestion:  https://collector.tracing.internal:4317 (gRPC)
                 https://collector.tracing.internal:4318 (HTTP)
Query API:       https://api.tracing.internal/v1
UI:              https://tracing.internal
```

---

## API Versioning Strategy

We use URL path versioning (`/v1/`, `/v2/`) for query APIs because:
- Easy to understand and implement
- Clear in logs and documentation
- Allows running multiple versions simultaneously

For span ingestion, we use protocol versioning (OpenTelemetry protocol versions).

**Backward Compatibility Rules:**

Non-breaking changes (no version bump):
- Adding new optional fields
- Adding new endpoints
- Adding new error codes

Breaking changes (require new version):
- Removing fields
- Changing field types
- Changing endpoint paths

---

## Span Ingestion API (OpenTelemetry Protocol)

### gRPC Endpoint: ExportSpans

**Protocol:** OpenTelemetry Protocol (OTLP) over gRPC

**Endpoint:** `/opentelemetry.proto.collector.trace.v1.TraceService/Export`

**Request:**

```protobuf
message ExportTraceServiceRequest {
  repeated ResourceSpans resource_spans = 1;
}

message ResourceSpans {
  Resource resource = 1;
  repeated InstrumentationLibrarySpans instrumentation_library_spans = 2;
}

message Span {
  bytes trace_id = 1;                    // 16 bytes
  bytes span_id = 2;                     // 8 bytes
  string trace_state = 3;                // W3C trace state
  bytes parent_span_id = 4;              // 8 bytes (empty if root)
  string name = 5;                       // Operation name
  int32 kind = 6;                        // Span kind (SERVER, CLIENT, etc.)
  fixed64 start_time_unix_nano = 7;      // Start timestamp
  fixed64 end_time_unix_nano = 8;        // End timestamp
  repeated KeyValue attributes = 9;      // Tags
  repeated SpanEvent events = 10;        // Logs
  repeated SpanLink links = 11;          // Links to other spans
  int32 status = 12;                     // Status code
}
```

**Response:**

```protobuf
message ExportTraceServiceResponse {
  // Empty for success, error details in gRPC status
}
```

**Error Handling:**

- gRPC status codes: OK, INVALID_ARGUMENT, RESOURCE_EXHAUSTED, UNAVAILABLE
- Rate limiting: RESOURCE_EXHAUSTED with retry_after metadata

---

## Query API

### 1. Get Trace by ID

**Endpoint:** `GET /v1/traces/{trace_id}`

**Purpose:** Retrieve complete trace by trace ID

**Request:**

```http
GET /v1/traces/abc123def45678901234567890123456 HTTP/1.1
Host: api.tracing.internal
Authorization: Bearer <token>
```

**Response:** `200 OK`

```json
{
  "trace_id": "abc123def45678901234567890123456",
  "spans": [
    {
      "trace_id": "abc123def45678901234567890123456",
      "span_id": "def789abc1234567",
      "parent_span_id": null,
      "operation_name": "GET /orders",
      "service_name": "api-gateway",
      "start_time": 1705312245000000,
      "duration": 500000000,
      "tags": {
        "http.method": "GET",
        "http.status_code": "200",
        "user.id": "user_123"
      },
      "logs": [
        {
          "timestamp": 1705312245100000,
          "fields": {
            "event": "database.query",
            "query": "SELECT * FROM orders"
          }
        }
      ],
      "status": "OK"
    },
    {
      "trace_id": "abc123def45678901234567890123456",
      "span_id": "abc456def7890123",
      "parent_span_id": "def789abc1234567",
      "operation_name": "getOrder",
      "service_name": "order-service",
      "start_time": 1705312245100000,
      "duration": 350000000,
      "tags": {
        "db.statement": "SELECT * FROM orders",
        "db.type": "postgresql"
      },
      "status": "OK"
    }
  ],
  "start_time": 1705312245000000,
  "duration": 500000000,
  "span_count": 2
}
```

### 2. Search Traces

**Endpoint:** `GET /v1/traces`

**Purpose:** Search traces by service, operation, tags, or time range

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| service_name | string | Filter by service name |
| operation_name | string | Filter by operation name |
| tag | string | Key-value tag filter (format: key=value) |
| min_duration | integer | Minimum duration in microseconds |
| max_duration | integer | Maximum duration in microseconds |
| start_time | integer | Start time (Unix timestamp in microseconds) |
| end_time | integer | End time (Unix timestamp in microseconds) |
| limit | integer | Maximum number of traces (default: 100, max: 1000) |
| offset | integer | Pagination offset |

**Request:**

```http
GET /v1/traces?service_name=order-service&tag=http.status_code=500&start_time=1705312245000000&end_time=1705312246000000&limit=50 HTTP/1.1
Host: api.tracing.internal
Authorization: Bearer <token>
```

**Response:** `200 OK`

```json
{
  "traces": [
    {
      "trace_id": "abc123...",
      "service_name": "order-service",
      "operation_name": "createOrder",
      "start_time": 1705312245100000,
      "duration": 2000000,
      "span_count": 5,
      "has_error": true
    }
  ],
  "total": 42,
  "limit": 50,
  "offset": 0
}
```

### 3. Get Service List

**Endpoint:** `GET /v1/services`

**Purpose:** List all services that have sent traces

**Request:**

```http
GET /v1/services HTTP/1.1
Host: api.tracing.internal
Authorization: Bearer <token>
```

**Response:** `200 OK`

```json
{
  "services": [
    "api-gateway",
    "order-service",
    "payment-service",
    "inventory-service"
  ]
}
```

### 4. Get Service Operations

**Endpoint:** `GET /v1/services/{service_name}/operations`

**Purpose:** List operations for a service

**Request:**

```http
GET /v1/services/order-service/operations HTTP/1.1
Host: api.tracing.internal
Authorization: Bearer <token>
```

**Response:** `200 OK`

```json
{
  "service_name": "order-service",
  "operations": [
    "createOrder",
    "getOrder",
    "updateOrder",
    "cancelOrder"
  ]
}
```

### 5. Get Service Metrics

**Endpoint:** `GET /v1/services/{service_name}/metrics`

**Purpose:** Get latency and error metrics for a service

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| start_time | integer | Start time (Unix timestamp) |
| end_time | integer | End time (Unix timestamp) |
| operation_name | string | Filter by operation (optional) |

**Request:**

```http
GET /v1/services/order-service/metrics?start_time=1705312245&end_time=1705312246 HTTP/1.1
Host: api.tracing.internal
Authorization: Bearer <token>
```

**Response:** `200 OK`

```json
{
  "service_name": "order-service",
  "period": {
    "start_time": 1705312245,
    "end_time": 1705312246
  },
  "metrics": {
    "request_count": 1000,
    "error_count": 50,
    "error_rate": 0.05,
    "latency": {
      "p50": 100000,
      "p95": 500000,
      "p99": 1000000,
      "p99_9": 2000000
    }
  },
  "operations": [
    {
      "operation_name": "createOrder",
      "request_count": 500,
      "error_count": 30,
      "latency": {
        "p50": 150000,
        "p95": 600000,
        "p99": 1200000
      }
    }
  ]
}
```

---

## Pagination Strategy

We use **offset-based pagination** for trace searches because:
- Simpler to implement
- Acceptable for internal tooling (not user-facing)
- Total count is available for UI display

**Limitations:**
- Offset-based pagination can be slow for large offsets
- For very large result sets, consider cursor-based pagination

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
| Span Ingestion | 10,000 per service |
| Trace Query | 100 |
| Metrics Query | 200 |
| Service List | 60 |

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
| 400 | INVALID_TRACE_ID | Trace ID format is invalid |
| 400 | INVALID_TIME_RANGE | Time range is invalid |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | TRACE_NOT_FOUND | Trace not found |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

**Error Response Examples:**

```json
// 404 Not Found - Trace not found
{
  "error": {
    "code": "TRACE_NOT_FOUND",
    "message": "Trace not found",
    "details": {
      "trace_id": "abc123...",
      "reason": "Trace does not exist or has expired"
    },
    "request_id": "req_abc123"
  }
}

// 400 Bad Request - Invalid time range
{
  "error": {
    "code": "INVALID_TIME_RANGE",
    "message": "Time range is invalid",
    "details": {
      "field": "end_time",
      "reason": "end_time must be after start_time"
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
      "limit": 100,
      "remaining": 0,
      "reset_at": "2024-01-15T11:00:00Z"
    },
    "request_id": "req_def456"
  }
}
```

---

## Authentication/Authorization

**Span Ingestion:**
- API key authentication (per service)
- Keys stored in secure vault
- Rate limiting per API key

**Query API:**
- OAuth 2.0 / JWT Bearer token
- RBAC: Read-only, Admin roles
- Audit logging for all queries

---

## Idempotency

Span ingestion is idempotent by design:
- Spans with same trace_id + span_id are deduplicated
- Duplicate spans are ignored (based on trace_id + span_id + start_time)

Query APIs are idempotent (GET requests).

---

## Summary

| Aspect | Decision |
|--------|----------|
| Span Ingestion | OpenTelemetry Protocol (OTLP) over gRPC and HTTP |
| Query API | RESTful HTTP API with JSON |
| Authentication | API keys (ingestion), OAuth 2.0/JWT (query) |
| Pagination | Offset-based (acceptable for internal tooling) |
| Rate Limiting | Per-service for ingestion, per-user for queries |
| Error Model | Standard error envelope with codes and details |


