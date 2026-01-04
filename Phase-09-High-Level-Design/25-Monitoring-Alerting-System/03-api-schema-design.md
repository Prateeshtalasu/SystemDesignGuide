# Monitoring & Alerting System - API & Schema Design

## API Design Philosophy

The monitoring system has two types of APIs:

1. **Metrics Ingestion API**: High-throughput endpoint for services to send metrics (Prometheus, StatsD, custom protocols)
2. **Query API**: RESTful API for querying metrics and managing alerts (for dashboards and integrations)

---

## Base URL Structure

```
Metrics Ingestion:  https://collector.monitoring.internal:9091 (Prometheus Pushgateway)
                    https://collector.monitoring.internal:8125 (StatsD UDP)
                    https://collector.monitoring.internal:8080 (Custom HTTP)
Query API:          https://api.monitoring.internal/v1
Dashboard:          https://monitoring.internal
```

---

## Metrics Ingestion API

### Prometheus Protocol (Pull Model)

**Endpoint:** `/metrics`

Services expose metrics endpoint, collector scrapes periodically.

**Response Format:**

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 12345
http_requests_total{method="POST",status="200"} 6789
http_requests_total{method="GET",status="500"} 12

# HELP http_request_duration_seconds HTTP request duration in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET",le="0.1"} 10000
http_request_duration_seconds_bucket{method="GET",le="0.5"} 15000
http_request_duration_seconds_bucket{method="GET",le="1.0"} 18000
http_request_duration_seconds_bucket{method="GET",le="+Inf"} 20000
http_request_duration_seconds_sum{method="GET"} 8500
http_request_duration_seconds_count{method="GET"} 20000
```

### StatsD Protocol (Push Model)

**Endpoint:** UDP `collector.monitoring.internal:8125`

**Format:**

```
service.request.count:1|c
service.request.latency:150|ms
service.active_users:42|g
service.error.rate:0.01|h
```

### Custom HTTP API (Push Model)

**Endpoint:** `POST /v1/metrics`

**Request:**

```json
{
  "metrics": [
    {
      "name": "http_requests_total",
      "type": "counter",
      "value": 1,
      "labels": {
        "method": "GET",
        "status": "200",
        "service": "order-service"
      },
      "timestamp": 1705312245000
    },
    {
      "name": "http_request_duration_seconds",
      "type": "histogram",
      "value": 0.15,
      "labels": {
        "method": "GET",
        "service": "order-service"
      },
      "timestamp": 1705312245000
    }
  ]
}
```

**Response:** `202 Accepted`

```json
{
  "status": "accepted",
  "metrics_received": 2,
  "timestamp": 1705312245100
}
```

---

## Query API

### 1. Query Metrics

**Endpoint:** `GET /v1/query`

**Purpose:** Execute a metric query (PromQL-like)

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| query | string | PromQL query expression |
| time | integer | Evaluation timestamp (Unix timestamp) |

**Request:**

```http
GET /v1/query?query=http_requests_total{service="order-service"}[5m]&time=1705312245 HTTP/1.1
Host: api.monitoring.internal
Authorization: Bearer <token>
```

**Response:** `200 OK`

```json
{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
        "metric": {
          "method": "GET",
          "status": "200",
          "service": "order-service"
        },
        "values": [
          [1705312200, "12345"],
          [1705312230, "12456"],
          [1705312260, "12567"]
        ]
      }
    ]
  }
}
```

### 2. Query Range

**Endpoint:** `GET /v1/query_range`

**Purpose:** Query metrics over a time range

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| query | string | PromQL query expression |
| start | integer | Start timestamp |
| end | integer | End timestamp |
| step | string | Query resolution step (e.g., "15s", "1m") |

**Request:**

```http
GET /v1/query_range?query=rate(http_requests_total[5m])&start=1705312200&end=1705312260&step=15s HTTP/1.1
Host: api.monitoring.internal
Authorization: Bearer <token>
```

**Response:** `200 OK`

```json
{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
        "metric": {
          "service": "order-service"
        },
        "values": [
          [1705312200, "100.5"],
          [1705312215, "102.3"],
          [1705312230, "98.7"]
        ]
      }
    ]
  }
}
```

### 3. List Metric Names

**Endpoint:** `GET /v1/label/__name__/values`

**Purpose:** List all metric names

**Request:**

```http
GET /v1/label/__name__/values HTTP/1.1
Host: api.monitoring.internal
Authorization: Bearer <token>
```

**Response:** `200 OK`

```json
{
  "status": "success",
  "data": [
    "http_requests_total",
    "http_request_duration_seconds",
    "cpu_usage_percent",
    "memory_usage_bytes"
  ]
}
```

### 4. List Label Values

**Endpoint:** `GET /v1/label/{label_name}/values`

**Purpose:** List all values for a label

**Request:**

```http
GET /v1/label/service/values HTTP/1.1
Host: api.monitoring.internal
Authorization: Bearer <token>
```

**Response:** `200 OK`

```json
{
  "status": "success",
  "data": [
    "order-service",
    "payment-service",
    "inventory-service"
  ]
}
```

---

## Alerting API

### 1. Create Alert Rule

**Endpoint:** `POST /v1/alerts/rules`

**Purpose:** Create a new alert rule

**Request:**

```http
POST /v1/alerts/rules HTTP/1.1
Host: api.monitoring.internal
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "high_error_rate",
  "expr": "rate(http_requests_total{status=~'5..'}[5m]) > 0.01",
  "for": "5m",
  "labels": {
    "severity": "critical",
    "service": "order-service"
  },
  "annotations": {
    "summary": "High error rate detected",
    "description": "Error rate is {{ $value }} which is above threshold of 0.01"
  }
}
```

**Response:** `201 Created`

```json
{
  "id": "alert_rule_123",
  "name": "high_error_rate",
  "expr": "rate(http_requests_total{status=~'5..'}[5m]) > 0.01",
  "for": "5m",
  "labels": {
    "severity": "critical",
    "service": "order-service"
  },
  "annotations": {
    "summary": "High error rate detected",
    "description": "Error rate is {{ $value }} which is above threshold of 0.01"
  },
  "created_at": "2024-01-15T10:30:00Z"
}
```

### 2. List Alert Rules

**Endpoint:** `GET /v1/alerts/rules`

**Purpose:** List all alert rules

**Response:** `200 OK`

```json
{
  "rules": [
    {
      "id": "alert_rule_123",
      "name": "high_error_rate",
      "expr": "rate(http_requests_total{status=~'5..'}[5m]) > 0.01",
      "state": "firing",
      "active_at": "2024-01-15T10:35:00Z"
    }
  ]
}
```

### 3. List Active Alerts

**Endpoint:** `GET /v1/alerts`

**Purpose:** List currently firing alerts

**Response:** `200 OK`

```json
{
  "alerts": [
    {
      "id": "alert_456",
      "rule_id": "alert_rule_123",
      "labels": {
        "severity": "critical",
        "service": "order-service"
      },
      "annotations": {
        "summary": "High error rate detected",
        "description": "Error rate is 0.015 which is above threshold of 0.01"
      },
      "state": "firing",
      "active_at": "2024-01-15T10:35:00Z",
      "value": "0.015"
    }
  ]
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
| POST /v1/metrics | ✅ Yes | Idempotency-Key header (for custom HTTP API) |
| POST /v1/alerts/rules | ✅ Yes | Idempotency-Key header |
| PUT /v1/alerts/rules/{id} | ✅ Yes | Idempotency-Key header or version-based |
| DELETE /v1/alerts/rules/{id} | ✅ Yes | Safe to retry (idempotent by design) |
| POST /v1/alerts/silences | ✅ Yes | Idempotency-Key header |

**Implementation Example:**

```java
@RestController
@RequestMapping("/v1/alerts/rules")
public class AlertRuleController {
    
    @PostMapping
    public ResponseEntity<AlertRule> createAlertRule(
            @RequestBody CreateAlertRuleRequest request,
            @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {
        
        // Check for existing idempotency key
        if (idempotencyKey != null) {
            String cacheKey = "idempotency:" + idempotencyKey;
            AlertRule cached = redisTemplate.opsForValue().get(cacheKey);
            if (cached != null) {
                return ResponseEntity.status(HttpStatus.OK).body(cached);
            }
        }
        
        AlertRule rule = alertRuleService.createRule(request, idempotencyKey);
        
        // Cache response if idempotency key provided
        if (idempotencyKey != null) {
            String cacheKey = "idempotency:" + idempotencyKey;
            redisTemplate.opsForValue().set(cacheKey, rule, Duration.ofHours(24));
        }
        
        return ResponseEntity.status(HttpStatus.CREATED).body(rule);
    }
}
```

---

## Error Model

All error responses follow this standard envelope structure:

```json
{
  "status": "error",
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "field": "field_name",
      "reason": "Specific reason"
    }
  }
}
```

**Error Codes Reference:**

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | INVALID_QUERY | Query expression is invalid |
| 400 | INVALID_TIME_RANGE | Time range is invalid |
| 400 | INVALID_ALERT_RULE | Alert rule definition is invalid |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Resource not found |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

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
| Metrics Ingestion | 100,000 per service |
| Query API | 1,000 |
| Alert Management | 100 |

---

## Authentication/Authorization

**Metrics Ingestion:**
- API key authentication (per service)
- Keys stored in secure vault
- Rate limiting per API key

**Query API:**
- OAuth 2.0 / JWT Bearer token
- RBAC: Read-only, Admin roles
- Audit logging for all queries

---

## Summary

| Aspect | Decision |
|--------|----------|
| Metrics Ingestion | Prometheus (pull), StatsD (push), Custom HTTP (push) |
| Query API | RESTful HTTP API with PromQL-like queries |
| Authentication | API keys (ingestion), OAuth 2.0/JWT (query) |
| Rate Limiting | Per-service for ingestion, per-user for queries |
| Error Model | Standard error envelope with codes and details |

