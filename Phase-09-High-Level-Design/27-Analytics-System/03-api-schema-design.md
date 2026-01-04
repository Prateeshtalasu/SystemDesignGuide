# Analytics System - API & Schema Design

## API Design Philosophy

The Analytics System provides two main API surfaces:

1. **Event Ingestion API**: High-throughput endpoint for clients to send events
2. **Query API**: For analysts and dashboards to retrieve analytics data

The ingestion API prioritizes throughput and reliability, while the query API prioritizes flexibility and performance.

---

## Base URL Structure

```
Event Ingestion: https://events.analytics.com/v1
Query API: https://api.analytics.com/v1
```

---

## API Versioning Strategy

We use URL path versioning (`/v1/`, `/v2/`) because:
- Easy to understand and implement
- Clear in logs and documentation
- Allows running multiple versions simultaneously
- Clients can opt into new features gradually

**Backward Compatibility Rules:**

Non-breaking changes (no version bump):
- Adding new optional fields to events
- Adding new query endpoints
- Adding new optional query parameters
- Adding new error codes

Breaking changes (require new version):
- Removing fields from events
- Changing field types
- Changing required fields
- Changing endpoint paths
- Changing authentication mechanism

**Deprecation Policy:**
1. Announce deprecation 6 months in advance
2. Return `Deprecation: true` header
3. Maintain old version for 12 months after new version release
4. Provide migration guide and SDK updates

---

## Authentication & Authorization

### Event Ingestion

**API Key Authentication:**
```http
X-API-Key: <api_key>
```

API keys are scoped to:
- Project/application
- Rate limits
- Event types allowed

### Query API

**OAuth 2.0 / JWT Bearer Token:**
```http
Authorization: Bearer <access_token>
```

Tokens include permissions:
- Read access to specific projects
- Query limits
- Data export permissions

---

## Rate Limiting

### Event Ingestion Rate Limits

**Per API Key:**
```http
X-RateLimit-Limit: 100000
X-RateLimit-Remaining: 99950
X-RateLimit-Reset: 1640000000
```

**Rate Limit Tiers:**
- Free: 10K events/day
- Standard: 1M events/day
- Enterprise: Unlimited (with SLA)

**Rate Limit Strategy:**
- Token bucket algorithm
- Per API key
- Distributed rate limiting (Redis)
- Soft limit warnings at 80%

### Query API Rate Limits

**Per User:**
```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
```

**Rate Limit Tiers:**
- Analyst: 1K queries/hour
- Power User: 10K queries/hour
- Admin: 100K queries/hour

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
    "request_id": "req_123456",
    "retry_after": 60
  }
}
```

**Error Codes Reference:**

| HTTP Status | Error Code | Description | Retryable |
|-------------|------------|-------------|-----------|
| 400 | INVALID_EVENT | Event validation failed | No |
| 400 | INVALID_SCHEMA | Event schema mismatch | No |
| 400 | INVALID_QUERY | Query syntax error | No |
| 401 | UNAUTHORIZED | Authentication required | No |
| 403 | FORBIDDEN | Insufficient permissions | No |
| 404 | NOT_FOUND | Resource not found | No |
| 409 | DUPLICATE_EVENT | Event already processed | No |
| 413 | PAYLOAD_TOO_LARGE | Batch too large | No |
| 429 | RATE_LIMITED | Rate limit exceeded | Yes |
| 500 | INTERNAL_ERROR | Server error | Yes |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable | Yes |

**Error Response Examples:**

```json
// 400 Bad Request - Invalid event
{
  "error": {
    "code": "INVALID_EVENT",
    "message": "Event validation failed",
    "details": {
      "field": "userId",
      "reason": "userId is required and must be a string"
    },
    "request_id": "req_abc123"
  }
}

// 429 Rate Limited
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded. Please try again later.",
    "details": {
      "limit": 100000,
      "remaining": 0,
      "reset_at": "2024-01-15T11:00:00Z"
    },
    "request_id": "req_def456",
    "retry_after": 60
  }
}

// 503 Service Unavailable
{
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "Service temporarily unavailable. Please retry.",
    "details": {
      "reason": "High load, please retry in 30 seconds"
    },
    "request_id": "req_xyz789",
    "retry_after": 30
  }
}
```

---

## Event Ingestion API

### POST /v1/events

Ingest a single event or batch of events.

**Request Headers:**
```http
Content-Type: application/json
X-API-Key: <api_key>
X-Request-ID: <uuid>  # Optional, for tracing
```

**Single Event Request:**
```json
{
  "eventId": "evt_1234567890abcdef",
  "userId": "user_123",
  "eventType": "page_view",
  "timestamp": "2024-01-15T10:30:00Z",
  "properties": {
    "page": "/products",
    "referrer": "https://google.com",
    "device": "mobile"
  },
  "context": {
    "ip": "192.168.1.1",
    "userAgent": "Mozilla/5.0...",
    "sessionId": "sess_abc123"
  }
}
```

**Batch Event Request:**
```json
{
  "events": [
    {
      "eventId": "evt_1",
      "userId": "user_123",
      "eventType": "page_view",
      "timestamp": "2024-01-15T10:30:00Z",
      "properties": { "page": "/products" }
    },
    {
      "eventId": "evt_2",
      "userId": "user_123",
      "eventType": "click",
      "timestamp": "2024-01-15T10:30:05Z",
      "properties": { "element": "add_to_cart_button" }
    }
  ]
}
```

**Response (202 Accepted):**
```json
{
  "status": "accepted",
  "eventsProcessed": 2,
  "requestId": "req_abc123"
}
```

**Field Validation:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| eventId | string | Yes | Unique event identifier (UUID) |
| userId | string | No | User identifier |
| eventType | string | Yes | Event type (e.g., "page_view", "purchase") |
| timestamp | ISO 8601 | No | Event timestamp (defaults to now) |
| properties | object | No | Event-specific properties |
| context | object | No | Request context (IP, user agent, etc.) |

**Idempotency:**

Events are idempotent based on `eventId`. Duplicate `eventId` values are deduplicated:
- If event with same `eventId` exists: Return 409 Conflict
- If event is new: Process and return 202 Accepted

**Deduplication Window:** 24 hours

---

### POST /v1/events/batch

High-throughput batch ingestion endpoint (optimized for large batches).

**Request:**
```json
{
  "batchId": "batch_123456",
  "events": [
    // Up to 10,000 events
  ]
}
```

**Response:**
```json
{
  "status": "accepted",
  "batchId": "batch_123456",
  "eventsProcessed": 10000,
  "eventsRejected": 0,
  "requestId": "req_abc123"
}
```

**Constraints:**
- Max 10,000 events per batch
- Max 10 MB payload size
- Events processed asynchronously
- Partial failures reported in response

---

## Query API

### GET /v1/metrics/{metricName}

Get a pre-aggregated metric.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| startTime | ISO 8601 | Yes | Start of time range |
| endTime | ISO 8601 | Yes | End of time range |
| granularity | string | No | Time granularity (minute, hour, day) |
| dimensions | string | No | Comma-separated dimensions to group by |
| filters | JSON | No | Filter conditions |

**Example Request:**
```http
GET /v1/metrics/daily_active_users?startTime=2024-01-01T00:00:00Z&endTime=2024-01-31T23:59:59Z&granularity=day&dimensions=country,device
```

**Response:**
```json
{
  "metric": "daily_active_users",
  "startTime": "2024-01-01T00:00:00Z",
  "endTime": "2024-01-31T23:59:59Z",
  "granularity": "day",
  "data": [
    {
      "timestamp": "2024-01-01T00:00:00Z",
      "value": 1250000,
      "dimensions": {
        "country": "US",
        "device": "mobile"
      }
    },
    {
      "timestamp": "2024-01-01T00:00:00Z",
      "value": 850000,
      "dimensions": {
        "country": "US",
        "device": "desktop"
      }
    }
  ]
}
```

---

### POST /v1/query

Execute a custom SQL-like query on raw events or aggregates.

**Request:**
```json
{
  "query": "SELECT COUNT(*) as event_count, eventType FROM events WHERE timestamp >= '2024-01-01' AND timestamp < '2024-01-02' GROUP BY eventType",
  "format": "json",
  "timeout": 300
}
```

**Response:**
```json
{
  "queryId": "query_123456",
  "status": "completed",
  "executionTime": 2.5,
  "rows": 10,
  "data": [
    {
      "event_count": 5000000,
      "eventType": "page_view"
    },
    {
      "event_count": 1000000,
      "eventType": "click"
    }
  ]
}
```

**Query Limits:**
- Max execution time: 5 minutes
- Max result size: 100 MB
- Max rows returned: 1 million
- Queries are queued if system is busy

---

### GET /v1/events

Query raw events (with limitations).

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| userId | string | No | Filter by user ID |
| eventType | string | No | Filter by event type |
| startTime | ISO 8601 | Yes | Start of time range |
| endTime | ISO 8601 | Yes | End of time range |
| limit | integer | No | Max results (default 100, max 1000) |
| cursor | string | No | Pagination cursor |

**Example Request:**
```http
GET /v1/events?userId=user_123&eventType=purchase&startTime=2024-01-01T00:00:00Z&endTime=2024-01-31T23:59:59Z&limit=100
```

**Response:**
```json
{
  "events": [
    {
      "eventId": "evt_1",
      "userId": "user_123",
      "eventType": "purchase",
      "timestamp": "2024-01-15T10:30:00Z",
      "properties": {
        "productId": "prod_456",
        "amount": 29.99
      }
    }
  ],
  "nextCursor": "cursor_abc123",
  "hasMore": true
}
```

**Pagination:**
- Cursor-based pagination
- Cursor is opaque token
- Use `nextCursor` from response for next page
- Max 1000 events per request

---

## Pagination Strategy

We use **cursor-based pagination** because:
- Consistent results even when data changes
- Efficient for large datasets
- No offset performance issues

**Implementation:**
```json
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJ0aW1lc3RhbXAiOiIyMDI0LTAxLTE1VDEwOjMwOjAwWiIsImlkIjoiZXZ0XzEyMzQ1NiJ9",
    "hasMore": true,
    "limit": 100
  }
}
```

**Cursor Format:**
- Base64-encoded JSON
- Contains timestamp and last event ID
- Opaque to clients

---

## Input Validation

### Event Validation Rules

1. **eventId**: Required, must be valid UUID format
2. **eventType**: Required, must match registered schema
3. **timestamp**: Optional, must be valid ISO 8601, not in future
4. **properties**: Optional, must be valid JSON object
5. **properties size**: Max 10 KB per event

### Schema Validation

Events are validated against registered schemas:
- Schema defines required/optional fields
- Schema defines field types
- Schema defines allowed values (enums)
- Unknown fields are allowed (for flexibility)

**Schema Registration:**
```json
POST /v1/schemas
{
  "eventType": "purchase",
  "schema": {
    "properties": {
      "productId": { "type": "string", "required": true },
      "amount": { "type": "number", "required": true },
      "currency": { "type": "string", "enum": ["USD", "EUR"], "required": true }
    }
  }
}
```

---

## API Versioning Examples

**Version in URL:**
```
/v1/events
/v2/events
```

**Version Header (Alternative):**
```http
API-Version: 2
```

We prefer URL versioning for clarity.

---

## SDK Examples

### JavaScript SDK

```javascript
import { Analytics } from '@analytics/sdk';

const analytics = new Analytics({
  apiKey: 'your_api_key',
  endpoint: 'https://events.analytics.com/v1'
});

// Track event
analytics.track('page_view', {
  page: '/products',
  referrer: 'https://google.com'
});

// Batch events
analytics.trackBatch([
  { eventType: 'page_view', properties: { page: '/products' } },
  { eventType: 'click', properties: { element: 'button' } }
]);
```

### Java SDK

```java
import com.analytics.sdk.AnalyticsClient;

AnalyticsClient client = new AnalyticsClient.Builder()
    .apiKey("your_api_key")
    .endpoint("https://events.analytics.com/v1")
    .build();

// Track event
Event event = Event.builder()
    .eventType("page_view")
    .userId("user_123")
    .property("page", "/products")
    .build();

client.track(event);
```

---

## Rate Limiting Headers

Every response includes rate limit information:

```http
X-RateLimit-Limit: 100000
X-RateLimit-Remaining: 99950
X-RateLimit-Reset: 1640000000
```

**Rate Limit Exceeded Response (429):**
```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded",
    "details": {
      "limit": 100000,
      "remaining": 0,
      "reset_at": "2024-01-15T11:00:00Z"
    },
    "retry_after": 60
  }
}
```

---

## Idempotency for Event Ingestion

Events are idempotent based on `eventId`:

1. Client sends event with `eventId`
2. Server checks if `eventId` already processed
3. If exists: Return 409 Conflict with original response
4. If new: Process event, store `eventId`, return 202 Accepted

**Deduplication Storage:**
- Redis with 24-hour TTL
- Key: `event:{eventId}`
- Value: Processing status

**Retry Semantics:**
- Clients should retry on 5xx errors
- Use exponential backoff
- Same `eventId` can be retried (idempotent)

---

## Webhook Support (Optional)

For real-time event processing, clients can register webhooks:

```json
POST /v1/webhooks
{
  "url": "https://client.com/webhook",
  "eventTypes": ["purchase", "signup"],
  "secret": "webhook_secret"
}
```

**Webhook Payload:**
```json
{
  "event": {
    "eventId": "evt_123",
    "eventType": "purchase",
    "timestamp": "2024-01-15T10:30:00Z",
    "properties": { "amount": 29.99 }
  },
  "signature": "sha256=..."
}
```

---

## Common Headers

**Request Headers:**
```http
Content-Type: application/json
X-API-Key: <api_key>
X-Request-ID: <uuid>
X-Client-Version: 2.5.0
X-SDK-Version: java-1.2.3
```

**Response Headers:**
```http
Content-Type: application/json
X-RateLimit-Limit: 100000
X-RateLimit-Remaining: 99950
X-RateLimit-Reset: 1640000000
X-Request-ID: <uuid>
X-Processing-Time: 0.05
```

---

## API Best Practices

1. **Use Batch Endpoint**: Send events in batches (up to 10K) for better throughput
2. **Include eventId**: Always include `eventId` for idempotency
3. **Set Timestamp**: Include accurate `timestamp` for time-based analysis
4. **Validate Schema**: Register event schemas for validation
5. **Handle Rate Limits**: Implement exponential backoff for 429 responses
6. **Use Cursors**: Use cursor-based pagination for large result sets
7. **Monitor Quotas**: Track rate limit usage via response headers

---

## Next Steps

After API design, we'll design:
1. Data model and storage architecture
2. Event processing pipelines
3. Query execution engine
4. Real-time aggregation system


