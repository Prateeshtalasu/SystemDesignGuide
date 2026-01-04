# Distributed Rate Limiter - API & Schema Design

## API Design Philosophy

A distributed rate limiter is primarily an internal service, so APIs focus on:

1. **Rate limit checking**: Check if request should be allowed
2. **Configuration management**: Set rate limits per user/endpoint
3. **Monitoring**: Track rate limit usage and violations
4. **Administration**: Override limits, reset counters

---

## Base URL Structure

```
Internal API: https://ratelimiter.internal/api/v1
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

## Rate Limiting Headers

Every response includes rate limit information:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
X-RateLimit-Strategy: token_bucket
```

**Header Fields:**

| Header | Description | Example |
|--------|------------|---------|
| X-RateLimit-Limit | Maximum requests allowed | 1000 |
| X-RateLimit-Remaining | Remaining requests in window | 999 |
| X-RateLimit-Reset | Unix timestamp when limit resets | 1640000000 |
| X-RateLimit-Strategy | Algorithm used (token_bucket/sliding_window) | token_bucket |

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
| 400 | INVALID_LIMIT | Rate limit value is invalid |
| 400 | INVALID_STRATEGY | Rate limit strategy is invalid |
| 404 | NOT_FOUND | Rate limit config not found |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Rate limiter unavailable (fail open) |

**Error Response Examples:**

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded. 1000 requests per minute allowed.",
    "details": {
      "limit": 1000,
      "remaining": 0,
      "reset_at": 1640000000
    },
    "request_id": "req_123456"
  }
}
```

---

## Core API Endpoints

### 1. Check Rate Limit

**Endpoint:** `POST /v1/rate-limit/check`

**Purpose:** Check if a request should be allowed based on rate limits.

**Request:**

```json
{
  "user_id": "user_123",
  "endpoint": "/api/v1/users",
  "strategy": "token_bucket",
  "limit": 1000,
  "window_seconds": 60
}
```

**Request Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| user_id | string | Yes | User identifier |
| endpoint | string | Yes | API endpoint being accessed |
| strategy | string | No | Rate limit strategy (token_bucket/sliding_window) |
| limit | integer | No | Override limit for this check |
| window_seconds | integer | No | Override window duration |

**Response (200 OK - Allowed):**

```json
{
  "allowed": true,
  "limit": 1000,
  "remaining": 999,
  "reset_at": 1640000000,
  "strategy": "token_bucket"
}
```

**Response (429 Too Many Requests - Rate Limited):**

```json
{
  "allowed": false,
  "limit": 1000,
  "remaining": 0,
  "reset_at": 1640000000,
  "strategy": "token_bucket",
  "retry_after": 30
}
```

**Response Headers:**

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
X-RateLimit-Strategy: token_bucket
```

**Java Implementation Example:**

```java
@RestController
@RequestMapping("/v1/rate-limit")
public class RateLimitController {
    
    @Autowired
    private RateLimitService rateLimitService;
    
    @PostMapping("/check")
    public ResponseEntity<RateLimitResponse> checkRateLimit(
            @RequestBody RateLimitRequest request) {
        
        RateLimitResult result = rateLimitService.checkLimit(
            request.getUserId(),
            request.getEndpoint(),
            request.getStrategy(),
            request.getLimit(),
            request.getWindowSeconds()
        );
        
        RateLimitResponse response = RateLimitResponse.builder()
            .allowed(result.isAllowed())
            .limit(result.getLimit())
            .remaining(result.getRemaining())
            .resetAt(result.getResetAt())
            .strategy(result.getStrategy())
            .build();
        
        HttpHeaders headers = new HttpHeaders();
        headers.add("X-RateLimit-Limit", String.valueOf(result.getLimit()));
        headers.add("X-RateLimit-Remaining", String.valueOf(result.getRemaining()));
        headers.add("X-RateLimit-Reset", String.valueOf(result.getResetAt()));
        headers.add("X-RateLimit-Strategy", result.getStrategy());
        
        HttpStatus status = result.isAllowed() 
            ? HttpStatus.OK 
            : HttpStatus.TOO_MANY_REQUESTS;
        
        return ResponseEntity.status(status)
            .headers(headers)
            .body(response);
    }
}
```

---

### 2. Get Rate Limit Status

**Endpoint:** `GET /v1/rate-limit/status/{user_id}/{endpoint}`

**Purpose:** Get current rate limit status for a user/endpoint combination.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| user_id | string | User identifier |
| endpoint | string | API endpoint |

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| strategy | string | Rate limit strategy (optional) |

**Response (200 OK):**

```json
{
  "user_id": "user_123",
  "endpoint": "/api/v1/users",
  "limit": 1000,
  "remaining": 750,
  "reset_at": 1640000000,
  "strategy": "token_bucket",
  "usage_percentage": 25.0
}
```

---

### 3. Configure Rate Limit

**Endpoint:** `PUT /v1/rate-limit/config`

**Purpose:** Configure rate limit for a user/endpoint combination.

**Request:**

```json
{
  "user_id": "user_123",
  "endpoint": "/api/v1/users",
  "limit": 2000,
  "window_seconds": 60,
  "strategy": "token_bucket",
  "burst_capacity": 100
}
```

**Request Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| user_id | string | Yes | User identifier |
| endpoint | string | Yes | API endpoint |
| limit | integer | Yes | Maximum requests per window |
| window_seconds | integer | Yes | Time window in seconds |
| strategy | string | Yes | Rate limit strategy |
| burst_capacity | integer | No | Token bucket burst capacity |

**Response (200 OK):**

```json
{
  "user_id": "user_123",
  "endpoint": "/api/v1/users",
  "limit": 2000,
  "window_seconds": 60,
  "strategy": "token_bucket",
  "burst_capacity": 100,
  "updated_at": "2024-01-20T10:00:00Z"
}
```

---

### 4. Reset Rate Limit Counter

**Endpoint:** `POST /v1/rate-limit/reset`

**Purpose:** Reset rate limit counter for a user/endpoint (admin operation).

**Request:**

```json
{
  "user_id": "user_123",
  "endpoint": "/api/v1/users"
}
```

**Response (200 OK):**

```json
{
  "user_id": "user_123",
  "endpoint": "/api/v1/users",
  "reset_at": "2024-01-20T10:00:00Z"
}
```

---

### 5. Batch Check Rate Limits

**Endpoint:** `POST /v1/rate-limit/batch-check`

**Purpose:** Check multiple rate limits in a single request (for efficiency).

**Request:**

```json
{
  "checks": [
    {
      "user_id": "user_123",
      "endpoint": "/api/v1/users"
    },
    {
      "user_id": "user_456",
      "endpoint": "/api/v1/posts"
    }
  ]
}
```

**Response (200 OK):**

```json
{
  "results": [
    {
      "user_id": "user_123",
      "endpoint": "/api/v1/users",
      "allowed": true,
      "remaining": 999
    },
    {
      "user_id": "user_456",
      "endpoint": "/api/v1/posts",
      "allowed": false,
      "remaining": 0
    }
  ]
}
```

---

## Redis Schema

### Token Bucket Storage

**Key Format:**
```
bucket:{user_id}:{endpoint}
```

**Value Structure:**
```
{
  "tokens": 100,
  "capacity": 100,
  "refill_rate": 10,
  "last_refill": 1640000000000
}
```

**TTL:** Window duration (e.g., 60 seconds)

**Operations:**
- `GET bucket:{user_id}:{endpoint}` - Get current token count
- `SET bucket:{user_id}:{endpoint} {value} EX {ttl}` - Set token count
- `INCR bucket:{user_id}:{endpoint}` - Decrement token (atomic)

---

### Sliding Window Storage

**Key Format:**
```
window:{user_id}:{endpoint}
```

**Data Structure:** Redis Sorted Set (ZSET)

**Score:** Timestamp (milliseconds)
**Member:** Request ID (UUID)

**Operations:**
- `ZADD window:{user_id}:{endpoint} {timestamp} {request_id}` - Add request
- `ZREMRANGEBYSCORE window:{user_id}:{endpoint} 0 {now-window}` - Remove old requests
- `ZCARD window:{user_id}:{endpoint}` - Count requests in window

**TTL:** Window duration (e.g., 60 seconds)

---

### Rate Limit Configuration Storage

**Key Format:**
```
config:{user_id}:{endpoint}
```

**Value Structure:**
```json
{
  "limit": 1000,
  "window_seconds": 60,
  "strategy": "token_bucket",
  "burst_capacity": 100,
  "updated_at": "2024-01-20T10:00:00Z"
}
```

**TTL:** None (persistent configuration)

---

## Database Schema (PostgreSQL)

### Rate Limit Configuration Table

```sql
CREATE TABLE rate_limit_config (
    id BIGSERIAL PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    endpoint VARCHAR(500) NOT NULL,
    limit_value INTEGER NOT NULL,
    window_seconds INTEGER NOT NULL,
    strategy VARCHAR(50) NOT NULL,
    burst_capacity INTEGER,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(user_id, endpoint)
);

CREATE INDEX idx_rate_limit_config_user ON rate_limit_config(user_id);
CREATE INDEX idx_rate_limit_config_endpoint ON rate_limit_config(endpoint);
```

---

### Rate Limit Logs Table

```sql
CREATE TABLE rate_limit_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    endpoint VARCHAR(500) NOT NULL,
    allowed BOOLEAN NOT NULL,
    limit_value INTEGER NOT NULL,
    remaining INTEGER NOT NULL,
    request_id VARCHAR(255),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rate_limit_logs_user ON rate_limit_logs(user_id, created_at);
CREATE INDEX idx_rate_limit_logs_endpoint ON rate_limit_logs(endpoint, created_at);
CREATE INDEX idx_rate_limit_logs_created ON rate_limit_logs(created_at);
```

---

## Idempotency

Rate limit checks are **idempotent** by design:

- Same user + endpoint + time window = same result
- Multiple checks within same window don't change the outcome
- Counter increments are atomic (Redis INCR)

**Idempotency Key (Optional):**

For batch operations, include an idempotency key:

```json
{
  "idempotency_key": "batch_123",
  "checks": [...]
}
```

If the same idempotency key is used within 5 minutes, return cached result.

---

## Authentication & Authorization

**Internal Service:**
- Service-to-service authentication using mTLS
- API keys for admin operations
- No user-facing authentication required

**Admin Operations:**
- Require admin API key
- Rate limit configuration changes logged
- Audit trail for all admin actions

---

## Summary

| Component | Technology | Notes |
|-----------|------------|-------|
| API Protocol | REST | JSON over HTTP |
| Rate Limit Storage | Redis | Fast counters, TTL support |
| Configuration Storage | PostgreSQL | Persistent config |
| Logs Storage | PostgreSQL | Audit trail |
| Headers | Standard rate limit headers | X-RateLimit-* |
| Idempotency | Built-in | Atomic operations |
