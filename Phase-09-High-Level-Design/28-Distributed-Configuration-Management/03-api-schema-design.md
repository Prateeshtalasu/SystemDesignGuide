# Distributed Configuration Management - API & Schema Design

## API Design Philosophy

The Configuration Management API provides:
1. **Configuration CRUD**: Create, read, update, delete configurations
2. **Version Management**: View history, rollback, compare versions
3. **Real-Time Updates**: Subscribe to configuration changes
4. **Feature Flags**: Manage feature flags and A/B tests
5. **Secrets Management**: Securely store and retrieve secrets

---

## Base URL Structure

```
Configuration API: https://config.company.com/v1
WebSocket API: wss://config.company.com/v1/ws
```

---

## API Versioning Strategy

URL path versioning (`/v1/`, `/v2/`) for clarity and backward compatibility.

**Backward Compatibility:**
- Non-breaking: Adding optional fields, new endpoints
- Breaking: Removing fields, changing types, changing paths

**Deprecation Policy:**
- 6 months notice
- Maintain old version for 12 months

---

## Authentication & Authorization

**OAuth 2.0 / JWT Bearer Token:**
```http
Authorization: Bearer <access_token>
```

**RBAC Roles:**
- Admin: Full access
- Developer: Read/write own service configs
- Viewer: Read-only access

---

## Rate Limiting

**Per User:**
```http
X-RateLimit-Limit: 10000
X-RateLimit-Remaining: 9990
X-RateLimit-Reset: 1640000000
```

**Tiers:**
- Standard: 10K requests/hour
- Premium: 100K requests/hour

---

## Error Model

Standard error envelope:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message",
    "details": {},
    "request_id": "req_123"
  }
}
```

**Error Codes:**
- 400: INVALID_INPUT, INVALID_KEY, VALIDATION_FAILED
- 401: UNAUTHORIZED
- 403: FORBIDDEN
- 404: NOT_FOUND
- 409: CONFLICT (version conflict)
- 429: RATE_LIMITED
- 500: INTERNAL_ERROR
- 503: SERVICE_UNAVAILABLE

---

## Configuration API

### GET /v1/config/{key}

Get configuration value.

**Path Parameters:**
- `key`: Configuration key (e.g., `services.payment.database_url`)

**Query Parameters:**
- `version`: Optional version number
- `environment`: Optional environment (default: production)

**Response:**
```json
{
  "key": "services.payment.database_url",
  "value": "postgres://db.example.com:5432/payment",
  "version": 42,
  "environment": "production",
  "updatedAt": "2024-01-15T10:30:00Z",
  "updatedBy": "user@example.com"
}
```

### GET /v1/config/{key}/history

Get configuration version history.

**Response:**
```json
{
  "key": "services.payment.database_url",
  "versions": [
    {
      "version": 42,
      "value": "postgres://db.example.com:5432/payment",
      "updatedAt": "2024-01-15T10:30:00Z",
      "updatedBy": "user@example.com",
      "changeLog": "Updated database URL"
    },
    {
      "version": 41,
      "value": "postgres://old-db.example.com:5432/payment",
      "updatedAt": "2024-01-14T09:00:00Z",
      "updatedBy": "admin@example.com",
      "changeLog": "Initial configuration"
    }
  ]
}
```

### PUT /v1/config/{key}

Update configuration.

**Request:**
```json
{
  "value": "postgres://new-db.example.com:5432/payment",
  "environment": "production",
  "changeLog": "Updated database URL for new cluster",
  "validate": true
}
```

**Response:**
```json
{
  "key": "services.payment.database_url",
  "value": "postgres://new-db.example.com:5432/payment",
  "version": 43,
  "updatedAt": "2024-01-15T11:00:00Z",
  "updatedBy": "user@example.com"
}
```

**Idempotency:**
- Use `If-Match` header with version number
- Prevents concurrent updates

### POST /v1/config/{key}/rollback

Rollback to previous version.

**Request:**
```json
{
  "version": 41,
  "changeLog": "Rolling back due to issues"
}
```

### GET /v1/config/bulk

Bulk retrieve configurations.

**Query Parameters:**
- `keys`: Comma-separated keys
- `prefix`: Key prefix filter
- `environment`: Environment filter

**Response:**
```json
{
  "configs": [
    {
      "key": "services.payment.database_url",
      "value": "postgres://...",
      "version": 42
    },
    {
      "key": "services.payment.max_connections",
      "value": "100",
      "version": 15
    }
  ]
}
```

---

## Feature Flags API

### GET /v1/feature-flags/{flag}

Get feature flag value.

**Response:**
```json
{
  "flag": "new_checkout",
  "enabled": true,
  "percentage": 50,
  "targetUsers": ["user_123", "user_456"],
  "environment": "production"
}
```

### PUT /v1/feature-flags/{flag}

Update feature flag.

**Request:**
```json
{
  "enabled": true,
  "percentage": 75,
  "targetUsers": ["user_123"],
  "changeLog": "Increasing rollout to 75%"
}
```

---

## WebSocket API (Real-Time Updates)

### WebSocket Connection

**Connection:**
```
wss://config.company.com/v1/ws?token=<jwt_token>
```

**Subscribe Message:**
```json
{
  "type": "subscribe",
  "keys": ["services.payment.database_url", "services.payment.max_connections"]
}
```

**Update Message (Server → Client):**
```json
{
  "type": "update",
  "key": "services.payment.database_url",
  "value": "postgres://new-db.example.com:5432/payment",
  "version": 43,
  "timestamp": "2024-01-15T11:00:00Z"
}
```

**Unsubscribe Message:**
```json
{
  "type": "unsubscribe",
  "keys": ["services.payment.database_url"]
}
```

---

## Secrets API

### POST /v1/secrets/{key}

Store secret.

**Request:**
```json
{
  "value": "secret_value",
  "environment": "production",
  "expiresAt": "2024-12-31T23:59:59Z"
}
```

**Response:**
```json
{
  "key": "services.payment.api_key",
  "version": 1,
  "createdAt": "2024-01-15T10:30:00Z",
  "expiresAt": "2024-12-31T23:59:59Z"
}
```

**Note:** Secret value is never returned in responses.

### GET /v1/secrets/{key}

Retrieve secret (decrypted).

**Response:**
```json
{
  "key": "services.payment.api_key",
  "value": "decrypted_secret",
  "version": 1
}
```

**Access Control:** Only authorized services can retrieve secrets.

---

## Pagination

**Cursor-based pagination:**

```json
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJ2ZXJzaW9uIjo0Miwia2V5Ijoic2VydmljZXMucGF5bWVudC4uLiJ9",
    "hasMore": true,
    "limit": 100
  }
}
```

---

## Input Validation

**Key Validation:**
- Must match pattern: `^[a-z0-9._-]+$`
- Max length: 256 characters
- Must be hierarchical (e.g., `services.payment.database_url`)

**Value Validation:**
- Max size: 10 KB
- JSON validation (if JSON type)
- Schema validation (if schema provided)

---

## Idempotency

**For Updates:**
- Use `If-Match` header with version number
- Prevents concurrent updates
- Returns 409 Conflict if version mismatch

**Example:**
```http
PUT /v1/config/services.payment.database_url
If-Match: 42
Content-Type: application/json

{
  "value": "postgres://new-db.example.com:5432/payment"
}
```

---

## Common Headers

**Request:**
```http
Authorization: Bearer <token>
If-Match: <version>
X-Request-ID: <uuid>
```

**Response:**
```http
Content-Type: application/json
X-RateLimit-Limit: 10000
X-RateLimit-Remaining: 9990
X-RateLimit-Reset: 1640000000
X-Request-ID: <uuid>
ETag: <version>
```

---

## SDK Examples

### Java SDK

```java
ConfigClient client = new ConfigClient.Builder()
    .endpoint("https://config.company.com/v1")
    .token("your_token")
    .build();

// Get config
String dbUrl = client.getConfig("services.payment.database_url");

// Update config
client.updateConfig("services.payment.database_url", 
    "postgres://new-db.example.com:5432/payment");

// Subscribe to updates
client.subscribe("services.payment.database_url", (key, value) -> {
    System.out.println("Config updated: " + key + " = " + value);
});
```

---

## Next Steps

After API design, we'll design:
1. Data model and storage architecture
2. Real-time update propagation
3. Caching strategy
4. Versioning and rollback mechanisms

