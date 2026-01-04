# URL Shortener - API & Schema Design

## API Design Philosophy

Before designing APIs, understand these principles:

1. **RESTful conventions**: Use HTTP methods semantically (GET for reads, POST for creates)
2. **Consistency**: Same patterns across all endpoints
3. **Idempotency**: Safe to retry failed requests
4. **Versioning**: Plan for API evolution
5. **Security**: Authentication, rate limiting, input validation

---

## Base URL Structure

```
Production: https://api.tinyurl.com/v1
Staging:    https://api.staging.tinyurl.com/v1
```

### API Versioning Strategy

We use URL path versioning (`/v1/`, `/v2/`) because:

- Easy to understand and implement
- Clear in logs and documentation
- Allows running multiple versions simultaneously

Alternatives considered:

- Header versioning (`Accept: application/vnd.tinyurl.v1+json`): Harder to test in browser
- Query parameter (`?version=1`): Feels hacky, caching issues

---

## Authentication

### API Key Authentication

For programmatic access, we use API keys:

```http
GET /v1/urls/abc123
Authorization: Bearer api_key_xxxxxxxxxxxxx
```

### Rate Limiting Headers

Every response includes rate limit information:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
```

### Rate Limits by Tier

| Tier       | Create URLs/hour | Redirects/hour | Analytics/hour |
| ---------- | ---------------- | -------------- | -------------- |
| Free       | 100              | Unlimited      | 1,000          |
| Pro        | 10,000           | Unlimited      | 100,000        |
| Enterprise | Unlimited        | Unlimited      | Unlimited      |

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
| 400 | INVALID_URL | URL format is invalid |
| 400 | URL_TOO_LONG | URL exceeds maximum length |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Resource not found |
| 409 | CONFLICT | Resource conflict (e.g., alias already exists) |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

**Error Response Examples:**

```json
// 400 Bad Request - Invalid URL
{
  "error": {
    "code": "INVALID_URL",
    "message": "The provided URL is not valid",
    "details": {
      "field": "original_url",
      "reason": "URL must start with http:// or https://"
    },
    "request_id": "req_abc123"
  }
}

// 409 Conflict - Alias already exists
{
  "error": {
    "code": "CONFLICT",
    "message": "Custom alias already exists",
    "details": {
      "field": "custom_alias",
      "reason": "The alias 'my-link' is already in use"
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

## Core API Endpoints

### 1. Create Short URL

**Endpoint:** `POST /v1/urls`

**Purpose:** Create a new short URL from a long URL

**Request:**

```http
POST /v1/urls HTTP/1.1
Host: api.tinyurl.com
Authorization: Bearer api_key_xxxxx
Content-Type: application/json

{
  "original_url": "https://example.com/very/long/path?with=params",
  "custom_alias": "my-link",        // Optional
  "expires_at": "2025-12-31T23:59:59Z",  // Optional
  "metadata": {                      // Optional
    "campaign": "summer-sale",
    "source": "email"
  }
}
```

**Response (201 Created):**

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: https://tiny.url/my-link

{
  "short_code": "my-link",
  "short_url": "https://tiny.url/my-link",
  "original_url": "https://example.com/very/long/path?with=params",
  "created_at": "2024-01-15T10:30:00Z",
  "expires_at": "2025-12-31T23:59:59Z",
  "is_custom": true,
  "metadata": {
    "campaign": "summer-sale",
    "source": "email"
  }
}
```

**Error Responses:**

```json
// 400 Bad Request - Invalid URL
{
  "error": {
    "code": "INVALID_URL",
    "message": "The provided URL is not valid",
    "details": {
      "field": "original_url",
      "reason": "URL must start with http:// or https://"
    }
  }
}

// 409 Conflict - Custom alias already taken
{
  "error": {
    "code": "ALIAS_TAKEN",
    "message": "The custom alias 'my-link' is already in use",
    "suggestion": "my-link-2024"
  }
}

// 429 Too Many Requests - Rate limited
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded. Try again in 60 seconds",
    "retry_after": 60
  }
}
```

**Input Validation Rules:**

| Field        | Validation                                                 |
| ------------ | ---------------------------------------------------------- |
| original_url | Required, valid HTTP/HTTPS URL, max 2048 chars             |
| custom_alias | Optional, 3-30 chars, alphanumeric + hyphens, no profanity |
| expires_at   | Optional, must be in future, max 10 years                  |

---

### 2. Redirect (Short URL Resolution)

**Endpoint:** `GET /{short_code}`

**Note:** This is on the redirect domain, not the API domain.

**Request:**

```http
GET /abc123 HTTP/1.1
Host: tiny.url
```

**Response (301 Moved Permanently):**

```http
HTTP/1.1 301 Moved Permanently
Location: https://example.com/very/long/path?with=params
Cache-Control: private, max-age=3600
```

**Why 301 vs 302?**

| Code | Name               | Use Case                                  |
| ---- | ------------------ | ----------------------------------------- |
| 301  | Moved Permanently  | SEO-friendly, browsers cache the redirect |
| 302  | Found (Temporary)  | When you might change the destination     |
| 307  | Temporary Redirect | Preserves HTTP method (POST stays POST)   |

We use **301** by default for SEO benefits, but allow users to configure 302.

**Error Response (404):**

```http
HTTP/1.1 404 Not Found
Content-Type: text/html

<!DOCTYPE html>
<html>
<head><title>Link Not Found</title></head>
<body>
  <h1>This link doesn't exist or has expired</h1>
  <p>The short URL you're looking for is not available.</p>
</body>
</html>
```

---

### 3. Get URL Details

**Endpoint:** `GET /v1/urls/{short_code}`

**Purpose:** Retrieve information about a short URL

**Request:**

```http
GET /v1/urls/abc123 HTTP/1.1
Host: api.tinyurl.com
Authorization: Bearer api_key_xxxxx
```

**Response (200 OK):**

```json
{
  "short_code": "abc123",
  "short_url": "https://tiny.url/abc123",
  "original_url": "https://example.com/very/long/path",
  "created_at": "2024-01-15T10:30:00Z",
  "expires_at": "2029-01-15T10:30:00Z",
  "is_custom": false,
  "click_count": 1542,
  "last_clicked_at": "2024-01-20T15:45:00Z",
  "metadata": {
    "campaign": "summer-sale"
  }
}
```

---

### 4. Update URL

**Endpoint:** `PATCH /v1/urls/{short_code}`

**Purpose:** Update URL properties (not the short code itself)

**Request:**

```http
PATCH /v1/urls/abc123 HTTP/1.1
Host: api.tinyurl.com
Authorization: Bearer api_key_xxxxx
Content-Type: application/json

{
  "original_url": "https://example.com/new/destination",
  "expires_at": "2026-12-31T23:59:59Z"
}
```

**Response (200 OK):**

```json
{
  "short_code": "abc123",
  "short_url": "https://tiny.url/abc123",
  "original_url": "https://example.com/new/destination",
  "updated_at": "2024-01-20T12:00:00Z",
  "expires_at": "2026-12-31T23:59:59Z"
}
```

---

### 5. Delete URL

**Endpoint:** `DELETE /v1/urls/{short_code}`

**Purpose:** Soft-delete a URL (stops redirects, keeps record)

**Request:**

```http
DELETE /v1/urls/abc123 HTTP/1.1
Host: api.tinyurl.com
Authorization: Bearer api_key_xxxxx
```

**Response (204 No Content):**

```http
HTTP/1.1 204 No Content
```

**Note:** We soft-delete to prevent short code reuse (security concern).

---

### 6. Get Analytics

**Endpoint:** `GET /v1/urls/{short_code}/analytics`

**Purpose:** Retrieve click analytics for a URL

**Request:**

```http
GET /v1/urls/abc123/analytics?period=7d HTTP/1.1
Host: api.tinyurl.com
Authorization: Bearer api_key_xxxxx
```

**Query Parameters:**

| Parameter   | Type   | Default | Description                         |
| ----------- | ------ | ------- | ----------------------------------- |
| period      | string | 7d      | Time period: 1d, 7d, 30d, 90d, 1y   |
| granularity | string | day     | Aggregation: hour, day, week, month |

**Response (200 OK):**

```json
{
  "short_code": "abc123",
  "period": {
    "start": "2024-01-13T00:00:00Z",
    "end": "2024-01-20T23:59:59Z"
  },
  "summary": {
    "total_clicks": 1542,
    "unique_visitors": 1203,
    "avg_clicks_per_day": 220
  },
  "clicks_by_day": [
    { "date": "2024-01-13", "clicks": 180 },
    { "date": "2024-01-14", "clicks": 210 },
    { "date": "2024-01-15", "clicks": 195 },
    { "date": "2024-01-16", "clicks": 245 },
    { "date": "2024-01-17", "clicks": 230 },
    { "date": "2024-01-18", "clicks": 255 },
    { "date": "2024-01-19", "clicks": 227 }
  ],
  "top_referrers": [
    { "referrer": "twitter.com", "clicks": 523 },
    { "referrer": "facebook.com", "clicks": 312 },
    { "referrer": "direct", "clicks": 289 }
  ],
  "top_countries": [
    { "country": "US", "clicks": 612 },
    { "country": "UK", "clicks": 234 },
    { "country": "DE", "clicks": 189 }
  ],
  "devices": {
    "mobile": 823,
    "desktop": 612,
    "tablet": 107
  }
}
```

---

### 7. List User's URLs

**Endpoint:** `GET /v1/urls`

**Purpose:** List all URLs created by the authenticated user

**Request:**

```http
GET /v1/urls?page=1&limit=20&sort=-created_at HTTP/1.1
Host: api.tinyurl.com
Authorization: Bearer api_key_xxxxx
```

**Query Parameters:**

| Parameter | Type   | Default     | Description                      |
| --------- | ------ | ----------- | -------------------------------- |
| page      | int    | 1           | Page number (1-indexed)          |
| limit     | int    | 20          | Items per page (max 100)         |
| sort      | string | -created_at | Sort field (prefix - for desc)   |
| status    | string | active      | Filter: active, expired, deleted |

**Response (200 OK):**

```json
{
  "data": [
    {
      "short_code": "abc123",
      "short_url": "https://tiny.url/abc123",
      "original_url": "https://example.com/page1",
      "created_at": "2024-01-15T10:30:00Z",
      "click_count": 1542
    },
    {
      "short_code": "xyz789",
      "short_url": "https://tiny.url/xyz789",
      "original_url": "https://example.com/page2",
      "created_at": "2024-01-10T08:00:00Z",
      "click_count": 892
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total_items": 156,
    "total_pages": 8,
    "has_next": true,
    "has_prev": false
  },
  "links": {
    "self": "/v1/urls?page=1&limit=20",
    "next": "/v1/urls?page=2&limit=20",
    "last": "/v1/urls?page=8&limit=20"
  }
}
```

---

## Entity Relationship Diagram

```
┌─────────────────┐       ┌─────────────────┐
│     users       │       │    api_keys     │
├─────────────────┤       ├─────────────────┤
│ id (PK)         │───┐   │ id (PK)         │
│ email           │   │   │ user_id (FK)    │──┐
│ password_hash   │   │   │ key_hash        │  │
│ tier            │   │   │ permissions     │  │
│ created_at      │   │   │ created_at      │  │
└─────────────────┘   │   └─────────────────┘  │
                      │                        │
                      │   ┌────────────────────┘
                      │   │
                      ▼   ▼
                ┌─────────────────┐
                │      urls       │
                ├─────────────────┤
                │ short_code (PK) │
                │ original_url    │
                │ user_id (FK)    │
                │ api_key_id (FK) │
                │ created_at      │
                │ expires_at      │
                │ click_count     │
                │ metadata        │
                └────────┬────────┘
                         │
                         │ 1:N
                         ▼
                ┌─────────────────┐       ┌─────────────────┐
                │     clicks      │       │   daily_stats   │
                ├─────────────────┤       ├─────────────────┤
                │ id (PK)         │       │ short_code (PK) │
                │ short_code (FK) │       │ date (PK)       │
                │ clicked_at      │       │ click_count     │
                │ ip_hash         │       │ unique_visitors │
                │ referrer        │       │ top_referrers   │
                │ country_code    │       │ country_breakdown│
                └─────────────────┘       └─────────────────┘
```

---

## Short Code Generation Strategy

### Option 1: Counter-Based (Chosen)

Use a global counter and convert to base62:

```java
public class ShortCodeGenerator {
    private static final String BASE62 =
        "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";

    private final AtomicLong counter;

    public ShortCodeGenerator(long startValue) {
        this.counter = new AtomicLong(startValue);
    }

    public String generate() {
        long id = counter.incrementAndGet();
        return toBase62(id);
    }

    private String toBase62(long num) {
        StringBuilder sb = new StringBuilder();
        while (num > 0) {
            sb.insert(0, BASE62.charAt((int)(num % 62)));
            num /= 62;
        }
        // Pad to minimum 6 characters
        while (sb.length() < 6) {
            sb.insert(0, '0');
        }
        return sb.toString();
    }
}
```

**Pros:**

- Guaranteed unique
- Predictable, sequential
- No collision handling needed

**Cons:**

- Sequential codes are guessable (security concern)
- Single point of failure (counter)

**Mitigation:** Shuffle bits or add random offset to obscure sequence.

### Option 2: Random Generation

Generate random 6-character strings:

```java
public String generateRandom() {
    SecureRandom random = new SecureRandom();
    StringBuilder sb = new StringBuilder(6);
    for (int i = 0; i < 6; i++) {
        sb.append(BASE62.charAt(random.nextInt(62)));
    }
    return sb.toString();
}
```

**Pros:**

- Not guessable
- No coordination needed

**Cons:**

- Collision possible (birthday paradox)
- Need retry logic

### Option 3: Hash-Based

Hash the original URL and take first 6 characters:

```java
public String generateFromHash(String url) {
    String hash = DigestUtils.md5Hex(url);
    return toBase62(Long.parseLong(hash.substring(0, 12), 16))
           .substring(0, 6);
}
```

**Pros:**

- Same URL always gets same short code
- Idempotent

**Cons:**

- Collisions when hash prefixes match
- Can't have multiple short URLs for same destination

### Chosen Approach: Hybrid

1. Use counter-based for guaranteed uniqueness
2. Shuffle bits to prevent guessing
3. For custom aliases, validate and store directly

---

## Idempotency Model

### What is Idempotency?

An operation is **idempotent** if performing it multiple times has the same effect as performing it once. This is critical for handling retries, network failures, and duplicate requests.

### Idempotent Operations in URL Shortener

| Operation | Idempotent? | Mechanism |
|-----------|-------------|-----------|
| **Create URL** | ✅ Yes | Idempotency key or custom alias uniqueness |
| **Get URL** | ✅ Yes | Read-only, no side effects |
| **Delete URL** | ✅ Yes | DELETE is idempotent (safe to retry) |
| **Update URL** | ✅ Yes | Idempotency key or version-based |
| **Record click** | ⚠️ At-least-once | Deduplication by (short_code, timestamp, ip_hash) |

### Idempotency Implementation

**1. URL Creation with Idempotency Key:**

```java
@PostMapping("/v1/urls")
public ResponseEntity<UrlResponse> createUrl(
        @RequestBody CreateUrlRequest request,
        @RequestHeader("Idempotency-Key") String idempotencyKey) {
    
    // Check if request already processed
    String cachedResponse = redisTemplate.opsForValue()
        .get("idempotency:" + idempotencyKey);
    if (cachedResponse != null) {
        return ResponseEntity.ok(parseResponse(cachedResponse));
    }
    
    // Process request
    UrlResponse response = urlService.createUrl(request);
    
    // Cache response for 24 hours
    redisTemplate.opsForValue().set(
        "idempotency:" + idempotencyKey,
        serialize(response),
        Duration.ofHours(24)
    );
    
    return ResponseEntity.status(201).body(response);
}
```

**2. Custom Alias Idempotency:**

```java
// If same original_url + custom_alias → return existing URL
public UrlResponse createUrlWithAlias(String originalUrl, String customAlias) {
    // Check if alias already exists
    Url existing = urlRepository.findByCustomAlias(customAlias);
    if (existing != null) {
        // If same original_url → idempotent (return existing)
        if (existing.getOriginalUrl().equals(originalUrl)) {
            return UrlResponse.from(existing);
        }
        // If different original_url → conflict
        throw new AliasConflictException("Alias already taken");
    }
    
    // Create new URL
    return urlService.createUrl(originalUrl, customAlias);
}
```

**3. Click Event Deduplication:**

```java
// Deduplicate by (short_code, timestamp, ip_hash)
public void recordClick(ClickEvent event) {
    String dedupKey = String.format("click:%s:%d:%s",
        event.getShortCode(),
        event.getTimestamp() / 1000, // Round to second
        event.getIpHash()
    );
    
    // Set if absent (idempotent)
    Boolean isNew = redisTemplate.opsForValue()
        .setIfAbsent(dedupKey, "1", Duration.ofMinutes(5));
    
    if (Boolean.TRUE.equals(isNew)) {
        // New click, process it
        kafkaTemplate.send("click-events", event);
    }
    // Duplicate click, ignore
}
```

### Idempotency Key Generation

**Client-Side (Recommended):**
```javascript
// Generate UUID v4 for each request
const idempotencyKey = crypto.randomUUID();
fetch('/v1/urls', {
    headers: {
        'Idempotency-Key': idempotencyKey
    }
});
```

**Server-Side (Fallback):**
```java
// If client doesn't provide key, generate from request hash
String idempotencyKey = DigestUtils.sha256Hex(
    request.getOriginalUrl() + 
    request.getCustomAlias() + 
    request.getUserId()
);
```

### Duplicate Detection Window

| Operation | Deduplication Window | Storage |
|-----------|---------------------|---------|
| URL Creation | 24 hours | Redis |
| Click Events | 5 minutes | Redis |
| Analytics Updates | 1 hour | ClickHouse (UPSERT) |

---

## Consistency Model

### URL Creation: Strong Consistency

When creating a URL, we need strong consistency:

- Cannot have two URLs with same short code
- User must see their URL immediately after creation

**Implementation:**

- Single primary database for writes
- Read-after-write consistency via reading from primary

### URL Redirect: Eventual Consistency (Acceptable)

For redirects, slight staleness is acceptable:

- If URL was just deleted, a few extra redirects are okay
- Cache can serve stale data briefly

**Implementation:**

- Read from replicas or cache
- TTL-based cache invalidation

### Analytics: Eventual Consistency

Click counts don't need real-time accuracy:

- Aggregate asynchronously
- Update counts in batches

---

## API Error Codes Reference

| HTTP Status | Error Code      | Description                         |
| ----------- | --------------- | ----------------------------------- |
| 400         | INVALID_URL     | URL format is invalid               |
| 400         | INVALID_ALIAS   | Custom alias format is invalid      |
| 400         | ALIAS_TOO_SHORT | Alias must be at least 3 characters |
| 400         | ALIAS_PROFANITY | Alias contains prohibited words     |
| 401         | UNAUTHORIZED    | Missing or invalid API key          |
| 403         | FORBIDDEN       | API key lacks required permission   |
| 404         | NOT_FOUND       | Short URL doesn't exist             |
| 409         | ALIAS_TAKEN     | Custom alias already in use         |
| 410         | EXPIRED         | Short URL has expired               |
| 429         | RATE_LIMITED    | Too many requests                   |
| 500         | INTERNAL_ERROR  | Server error                        |

---

## Backward Compatibility

### API Versioning Rules

1. **Non-breaking changes** (no version bump):

   - Adding new optional fields
   - Adding new endpoints
   - Adding new error codes

2. **Breaking changes** (require new version):
   - Removing fields
   - Changing field types
   - Changing endpoint paths
   - Changing authentication

### Deprecation Policy

1. Announce deprecation 6 months in advance
2. Return `Deprecation` header: `Deprecation: true`
3. Return `Sunset` header: `Sunset: Sat, 01 Jan 2025 00:00:00 GMT`
4. Maintain old version for 12 months after new version release
