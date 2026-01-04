# Distributed Cache - API & Schema Design

## API Design Philosophy

A distributed cache is primarily an internal service, so APIs focus on:

1. **Cache operations**: GET, SET, DELETE operations
2. **Batch operations**: Efficient multi-key operations
3. **Cache management**: TTL management, invalidation
4. **Monitoring**: Cache statistics, health checks

---

## Base URL Structure

```
Internal API: https://cache.internal/api/v1
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
| 400 | INVALID_KEY | Cache key format is invalid |
| 400 | INVALID_TTL | TTL value is invalid |
| 404 | NOT_FOUND | Cache key not found |
| 413 | PAYLOAD_TOO_LARGE | Value exceeds maximum size |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Cache node unavailable |

**Error Response Examples:**

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Cache key 'user:123' not found",
    "details": {
      "key": "user:123"
    },
    "request_id": "req_123456"
  }
}
```

---

## Core API Endpoints

### 1. Get Cache Value

**Endpoint:** `GET /v1/cache/{key}`

**Purpose:** Retrieve a value from the cache.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| key | string | Cache key (URL encoded) |

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| format | string | Response format (json/binary) |

**Response (200 OK - Cache Hit):**

```json
{
  "key": "user:123",
  "value": {
    "id": "123",
    "name": "John Doe",
    "email": "john@example.com"
  },
  "ttl": 3600,
  "cached_at": "2024-01-20T10:00:00Z",
  "expires_at": "2024-01-20T11:00:00Z"
}
```

**Response (404 Not Found - Cache Miss):**

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Cache key 'user:123' not found",
    "details": {
      "key": "user:123"
    },
    "request_id": "req_123456"
  }
}
```

**Response Headers:**

```http
HTTP/1.1 200 OK
X-Cache-Hit: true
X-Cache-Node: node-3
X-Cache-TTL: 3600
X-Cache-Age: 1800
```

**Java Implementation Example:**

```java
@RestController
@RequestMapping("/v1/cache")
public class CacheController {
    
    @Autowired
    private CacheService cacheService;
    
    @GetMapping("/{key}")
    public ResponseEntity<CacheResponse> getCache(
            @PathVariable String key,
            @RequestParam(required = false) String format) {
        
        CacheResult result = cacheService.get(key);
        
        if (result.isHit()) {
            CacheResponse response = CacheResponse.builder()
                .key(key)
                .value(result.getValue())
                .ttl(result.getTtl())
                .cachedAt(result.getCachedAt())
                .expiresAt(result.getExpiresAt())
                .build();
            
            HttpHeaders headers = new HttpHeaders();
            headers.add("X-Cache-Hit", "true");
            headers.add("X-Cache-Node", result.getNodeId());
            headers.add("X-Cache-TTL", String.valueOf(result.getTtl()));
            headers.add("X-Cache-Age", String.valueOf(result.getAge()));
            
            return ResponseEntity.ok()
                .headers(headers)
                .body(response);
        } else {
            HttpHeaders headers = new HttpHeaders();
            headers.add("X-Cache-Hit", "false");
            
            return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .headers(headers)
                .body(null);
        }
    }
}
```

---

### 2. Set Cache Value

**Endpoint:** `PUT /v1/cache/{key}`

**Purpose:** Store a value in the cache.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| key | string | Cache key (URL encoded) |

**Request Body:**

```json
{
  "value": {
    "id": "123",
    "name": "John Doe",
    "email": "john@example.com"
  },
  "ttl": 3600,
  "overwrite": true
}
```

**Request Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| value | object | Yes | Value to cache (JSON or binary) |
| ttl | integer | No | Time to live in seconds (default: 3600) |
| overwrite | boolean | No | Overwrite existing value (default: true) |

**Response (200 OK):**

```json
{
  "key": "user:123",
  "cached": true,
  "ttl": 3600,
  "expires_at": "2024-01-20T11:00:00Z",
  "node": "node-3"
}
```

**Response (409 Conflict - Key Exists):**

```json
{
  "error": {
    "code": "KEY_EXISTS",
    "message": "Cache key 'user:123' already exists",
    "details": {
      "key": "user:123"
    },
    "request_id": "req_123456"
  }
}
```

---

### 3. Delete Cache Value

**Endpoint:** `DELETE /v1/cache/{key}`

**Purpose:** Remove a value from the cache.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| key | string | Cache key (URL encoded) |

**Response (200 OK):**

```json
{
  "key": "user:123",
  "deleted": true
}
```

**Response (404 Not Found):**

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Cache key 'user:123' not found",
    "details": {
      "key": "user:123"
    },
    "request_id": "req_123456"
  }
}
```

---

### 4. Batch Get Cache Values

**Endpoint:** `POST /v1/cache/batch-get`

**Purpose:** Retrieve multiple values in a single request.

**Request:**

```json
{
  "keys": [
    "user:123",
    "user:456",
    "user:789"
  ]
}
```

**Response (200 OK):**

```json
{
  "results": [
    {
      "key": "user:123",
      "value": {
        "id": "123",
        "name": "John Doe"
      },
      "hit": true
    },
    {
      "key": "user:456",
      "value": null,
      "hit": false
    },
    {
      "key": "user:789",
      "value": {
        "id": "789",
        "name": "Jane Smith"
      },
      "hit": true
    }
  ],
  "hit_rate": 0.67
}
```

---

### 5. Batch Set Cache Values

**Endpoint:** `POST /v1/cache/batch-set`

**Purpose:** Store multiple values in a single request.

**Request:**

```json
{
  "items": [
    {
      "key": "user:123",
      "value": {
        "id": "123",
        "name": "John Doe"
      },
      "ttl": 3600
    },
    {
      "key": "user:456",
      "value": {
        "id": "456",
        "name": "Jane Smith"
      },
      "ttl": 3600
    }
  ]
}
```

**Response (200 OK):**

```json
{
  "saved": 2,
  "failed": 0,
  "results": [
    {
      "key": "user:123",
      "saved": true,
      "node": "node-3"
    },
    {
      "key": "user:456",
      "saved": true,
      "node": "node-5"
    }
  ]
}
```

---

### 6. Invalidate Cache Pattern

**Endpoint:** `POST /v1/cache/invalidate`

**Purpose:** Invalidate cache keys matching a pattern.

**Request:**

```json
{
  "pattern": "user:*",
  "async": true
}
```

**Request Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pattern | string | Yes | Key pattern (supports wildcards) |
| async | boolean | No | Process asynchronously (default: false) |

**Response (200 OK):**

```json
{
  "pattern": "user:*",
  "invalidated": 1000,
  "async": true,
  "job_id": "invalidation_job_123"
}
```

---

### 7. Get Cache Statistics

**Endpoint:** `GET /v1/cache/stats`

**Purpose:** Get cache statistics (hit rate, size, etc.).

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| node | string | Specific node ID (optional) |

**Response (200 OK):**

```json
{
  "total_keys": 1000000,
  "total_size_bytes": 10737418240,
  "hit_rate": 0.85,
  "miss_rate": 0.15,
  "evictions": 5000,
  "nodes": [
    {
      "node_id": "node-1",
      "keys": 100000,
      "size_bytes": 1073741824,
      "hit_rate": 0.86
    },
    {
      "node_id": "node-2",
      "keys": 100000,
      "size_bytes": 1073741824,
      "hit_rate": 0.84
    }
  ]
}
```

---

## Redis Schema

### Cache Value Storage

**Key Format:**
```
{key}
```

**Value Structure:**
```
{
  "data": {...},
  "cached_at": 1640000000000,
  "ttl": 3600
}
```

**TTL:** As specified (default: 3600 seconds)

**Operations:**
- `GET {key}` - Get value
- `SET {key} {value} EX {ttl}` - Set value with TTL
- `DEL {key}` - Delete value
- `EXISTS {key}` - Check if key exists
- `TTL {key}` - Get remaining TTL

---

### Cache Metadata Storage

**Key Format:**
```
meta:{key}
```

**Value Structure:**
```json
{
  "cached_at": 1640000000000,
  "access_count": 100,
  "last_accessed": 1640000000000,
  "size_bytes": 1024
}
```

**TTL:** Same as cache value

---

### Cache Index (for pattern invalidation)

**Key Format:**
```
index:{pattern}
```

**Data Structure:** Redis Set

**Members:** Cache keys matching the pattern

**Operations:**
- `SADD index:{pattern} {key}` - Add key to index
- `SMEMBERS index:{pattern}` - Get all keys matching pattern
- `SREM index:{pattern} {key}` - Remove key from index

---

## Database Schema (PostgreSQL)

### Cache Statistics Table

```sql
CREATE TABLE cache_statistics (
    id BIGSERIAL PRIMARY KEY,
    node_id VARCHAR(255) NOT NULL,
    total_keys BIGINT NOT NULL,
    total_size_bytes BIGINT NOT NULL,
    hit_count BIGINT NOT NULL,
    miss_count BIGINT NOT NULL,
    eviction_count BIGINT NOT NULL,
    recorded_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_cache_statistics_node ON cache_statistics(node_id, recorded_at);
CREATE INDEX idx_cache_statistics_recorded ON cache_statistics(recorded_at);
```

---

### Cache Invalidation Logs Table

```sql
CREATE TABLE cache_invalidation_logs (
    id BIGSERIAL PRIMARY KEY,
    pattern VARCHAR(500) NOT NULL,
    keys_invalidated INTEGER NOT NULL,
    invalidated_by VARCHAR(255),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_cache_invalidation_pattern ON cache_invalidation_logs(pattern, created_at);
CREATE INDEX idx_cache_invalidation_created ON cache_invalidation_logs(created_at);
```

---

## Idempotency

Cache operations are **idempotent** by design:

- **GET**: Multiple GETs return same value (idempotent)
- **SET**: Setting same key+value multiple times = same result
- **DELETE**: Deleting non-existent key = no-op (idempotent)

**Idempotency Key (Optional):**

For batch operations, include an idempotency key:

```json
{
  "idempotency_key": "batch_123",
  "items": [...]
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
- Cache invalidation operations logged
- Audit trail for all admin actions

---

## Summary

| Component | Technology | Notes |
|-----------|------------|-------|
| API Protocol | REST | JSON over HTTP |
| Cache Storage | Redis | In-memory storage |
| Statistics Storage | PostgreSQL | Persistent stats |
| Key Format | String | URL encoded |
| TTL | Seconds | Configurable per key |
| Idempotency | Built-in | All operations idempotent |
