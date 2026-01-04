# Notification System - API & Schema Design

## 1. API Design Principles

### RESTful API Standards

```
Base URL: https://api.notifications.example.com/v1

Authentication: API Key or OAuth 2.0
Rate Limiting: 10,000 requests/minute per API key
Versioning: URL path versioning (/v1/, /v2/)
Content-Type: application/json
```

### Common Headers

```http
# Request Headers
Authorization: Bearer <api_key>
X-Request-ID: <uuid>
X-Idempotency-Key: <uuid>  # For duplicate prevention
Content-Type: application/json

# Response Headers
X-RateLimit-Limit: 10000
X-RateLimit-Remaining: 9999
X-RateLimit-Reset: 1609459200
X-Request-ID: <uuid>
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
- Changing authentication

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
| 400 | INVALID_TEMPLATE | Notification template not found |
| 400 | INVALID_CHANNEL | Notification channel not supported |
| 400 | INVALID_USER | User ID is invalid |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Notification or template not found |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

**Error Response Examples:**

```json
// 400 Bad Request - Invalid template
{
  "error": {
    "code": "INVALID_TEMPLATE",
    "message": "Notification template not found",
    "details": {
      "field": "template_id",
      "reason": "Template 'order_confirmation' does not exist"
    },
    "request_id": "req_abc123"
  }
}

// 400 Bad Request - Invalid channel
{
  "error": {
    "code": "INVALID_CHANNEL",
    "message": "Notification channel not supported",
    "details": {
      "field": "channels",
      "reason": "Channel 'sms' is not enabled for this account"
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
      "limit": 10000,
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

Clients must include `Idempotency-Key` header for all write operations (already shown in request headers):
```http
X-Idempotency-Key: <uuid-v4>
```

**Deduplication Storage:**

Idempotency keys are stored in Redis with 24-hour TTL:
```
Key: idempotency:{key}
Value: Serialized response
TTL: 24 hours
```

**Retry Semantics:**

1. Client sends request with X-Idempotency-Key
2. Server checks Redis for existing key
3. If found: Return cached response (same status code + body)
4. If not found: Process request, cache response, return result
5. Retries with same key within 24 hours return cached response

**Per-Endpoint Idempotency:**

| Endpoint | Idempotent? | Mechanism |
|----------|-------------|-----------|
| POST /v1/notifications | Yes | X-Idempotency-Key header (prevents duplicate notifications) |
| PUT /v1/templates/{id} | Yes | X-Idempotency-Key or version-based |
| DELETE /v1/templates/{id} | Yes | Safe to retry (idempotent by design) |

**Implementation Example:**

```java
@PostMapping("/v1/notifications")
public ResponseEntity<NotificationResponse> sendNotification(
        @RequestBody SendNotificationRequest request,
        @RequestHeader(value = "X-Idempotency-Key", required = false) String idempotencyKey) {
    
    // Check for existing idempotency key (prevents duplicate notifications)
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        String cachedResponse = redisTemplate.opsForValue().get(cacheKey);
        if (cachedResponse != null) {
            NotificationResponse response = objectMapper.readValue(cachedResponse, NotificationResponse.class);
            return ResponseEntity.status(response.getStatus()).body(response);
        }
    }
    
    // Send notification
    Notification notification = notificationService.sendNotification(request, idempotencyKey);
    NotificationResponse response = NotificationResponse.from(notification);
    
    // Cache response if idempotency key provided
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        redisTemplate.opsForValue().set(
            cacheKey, 
            objectMapper.writeValueAsString(response),
            Duration.ofHours(24)
        );
    }
    
    return ResponseEntity.status(202).body(response);
}
```

---

## 2. Send Notification API

### Send Single Notification

```http
POST /v1/notifications
Content-Type: application/json
X-Idempotency-Key: <uuid>

Request:
{
  "user_id": "user_123",
  "template_id": "order_confirmation",
  "channels": ["push", "email"],
  "data": {
    "order_id": "ORD-456",
    "order_total": "$99.99",
    "delivery_date": "2024-01-20"
  },
  "options": {
    "priority": "high",
    "ttl": 86400,
    "collapse_key": "order_update",
    "scheduled_at": null,
    "respect_quiet_hours": true
  }
}

Response: 202 Accepted
{
  "notification_id": "notif_abc123",
  "status": "queued",
  "channels": {
    "push": {
      "status": "queued",
      "device_count": 2
    },
    "email": {
      "status": "queued",
      "email": "user@example.com"
    }
  },
  "created_at": "2024-01-15T10:30:00Z"
}
```

### Send Bulk Notifications

```http
POST /v1/notifications/bulk
Content-Type: application/json

Request:
{
  "notifications": [
    {
      "user_id": "user_123",
      "template_id": "promo_alert",
      "data": { "discount": "20%" }
    },
    {
      "user_id": "user_456",
      "template_id": "promo_alert",
      "data": { "discount": "15%" }
    }
  ],
  "default_channels": ["push"],
  "default_options": {
    "priority": "normal",
    "ttl": 3600
  }
}

Response: 202 Accepted
{
  "batch_id": "batch_xyz789",
  "total": 2,
  "queued": 2,
  "failed": 0,
  "notifications": [
    {
      "notification_id": "notif_001",
      "user_id": "user_123",
      "status": "queued"
    },
    {
      "notification_id": "notif_002",
      "user_id": "user_456",
      "status": "queued"
    }
  ]
}
```

### Send to Segment

```http
POST /v1/notifications/segment
Content-Type: application/json

Request:
{
  "segment": {
    "filters": [
      { "field": "country", "operator": "eq", "value": "US" },
      { "field": "last_active", "operator": "gt", "value": "2024-01-01" },
      { "field": "subscription", "operator": "in", "value": ["premium", "pro"] }
    ]
  },
  "template_id": "new_feature_announcement",
  "channels": ["push", "email"],
  "data": {
    "feature_name": "Dark Mode",
    "launch_date": "2024-01-20"
  },
  "options": {
    "priority": "normal",
    "batch_size": 10000,
    "throttle_per_second": 50000
  }
}

Response: 202 Accepted
{
  "campaign_id": "camp_abc123",
  "status": "processing",
  "estimated_recipients": 5000000,
  "started_at": "2024-01-15T10:30:00Z"
}
```

### Send to Topic

```http
POST /v1/notifications/topic/{topic_id}
Content-Type: application/json

Request:
{
  "template_id": "breaking_news",
  "data": {
    "headline": "Major Update Released",
    "summary": "Check out the new features..."
  },
  "channels": ["push"],
  "options": {
    "priority": "high"
  }
}

Response: 202 Accepted
{
  "broadcast_id": "bcast_xyz",
  "topic_id": "sports_news",
  "subscriber_count": 1500000,
  "status": "sending"
}
```

---

## 3. Template Management API

### Create Template

```http
POST /v1/templates
Content-Type: application/json

Request:
{
  "name": "order_confirmation",
  "description": "Sent when order is confirmed",
  "channels": {
    "push": {
      "title": "Order Confirmed! 🎉",
      "body": "Your order {{order_id}} for {{order_total}} is confirmed.",
      "image_url": "https://cdn.example.com/order-confirmed.png",
      "action_url": "/orders/{{order_id}}"
    },
    "email": {
      "subject": "Order Confirmed - {{order_id}}",
      "html_template_id": "email_order_confirm_v2",
      "from_name": "Example Store",
      "from_email": "orders@example.com"
    },
    "sms": {
      "body": "Order {{order_id}} confirmed. Total: {{order_total}}. Track: {{tracking_url}}"
    }
  },
  "default_data": {
    "company_name": "Example Store"
  },
  "category": "transactional",
  "tags": ["orders", "confirmation"]
}

Response: 201 Created
{
  "id": "tmpl_abc123",
  "name": "order_confirmation",
  "version": 1,
  "channels": ["push", "email", "sms"],
  "created_at": "2024-01-15T10:30:00Z",
  "created_by": "user_admin"
}
```

### Get Template

```http
GET /v1/templates/{template_id}
Authorization: Bearer <api_key>

Response: 200 OK
{
  "id": "tmpl_abc123",
  "name": "order_confirmation",
  "description": "Sent when order is confirmed",
  "version": 3,
  "channels": {
    "push": { ... },
    "email": { ... },
    "sms": { ... }
  },
  "variables": ["order_id", "order_total", "tracking_url"],
  "category": "transactional",
  "stats": {
    "sent_30d": 150000,
    "delivered_30d": 147000,
    "opened_30d": 45000
  },
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-18T14:00:00Z"
}
```

### List Templates

```http
GET /v1/templates
Authorization: Bearer <api_key>

Query Parameters:
- category: string (transactional, marketing, system)
- channel: string (push, email, sms)
- search: string
- limit: integer (default: 20)
- cursor: string

Response: 200 OK
{
  "templates": [
    {
      "id": "tmpl_abc123",
      "name": "order_confirmation",
      "channels": ["push", "email", "sms"],
      "category": "transactional",
      "updated_at": "2024-01-18T14:00:00Z"
    }
  ],
  "cursor": "eyJsYXN0X2lkIjoidG1wbF9hYmMxMjMifQ==",
  "has_more": true
}
```

---

## 4. Device Token API

### Register Device Token

```http
POST /v1/devices
Content-Type: application/json

Request:
{
  "user_id": "user_123",
  "token": "fcm_token_abc123...",
  "platform": "android",  // ios, android, web
  "app_version": "2.5.0",
  "device_model": "Pixel 7",
  "os_version": "Android 14",
  "timezone": "America/New_York",
  "language": "en"
}

Response: 201 Created
{
  "device_id": "dev_xyz789",
  "user_id": "user_123",
  "platform": "android",
  "token_hash": "abc123...",  // Partial hash for reference
  "created_at": "2024-01-15T10:30:00Z"
}
```

### Update Device Token

```http
PUT /v1/devices/{device_id}
Content-Type: application/json

Request:
{
  "token": "new_fcm_token_xyz...",
  "app_version": "2.6.0"
}

Response: 200 OK
{
  "device_id": "dev_xyz789",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

### Unregister Device

```http
DELETE /v1/devices/{device_id}
Authorization: Bearer <api_key>

Response: 204 No Content
```

---

## 5. User Preferences API

### Get User Preferences

```http
GET /v1/users/{user_id}/preferences
Authorization: Bearer <api_key>

Response: 200 OK
{
  "user_id": "user_123",
  "channels": {
    "push": {
      "enabled": true,
      "categories": {
        "marketing": false,
        "transactional": true,
        "social": true
      }
    },
    "email": {
      "enabled": true,
      "frequency": "daily_digest",
      "categories": {
        "marketing": true,
        "transactional": true,
        "newsletter": false
      }
    },
    "sms": {
      "enabled": true,
      "categories": {
        "security": true,
        "marketing": false
      }
    }
  },
  "quiet_hours": {
    "enabled": true,
    "start": "22:00",
    "end": "08:00",
    "timezone": "America/New_York",
    "override_for_urgent": true
  },
  "topics": {
    "subscribed": ["product_updates", "weekly_digest"],
    "unsubscribed": ["promotions"]
  },
  "updated_at": "2024-01-15T10:30:00Z"
}
```

### Update User Preferences

```http
PATCH /v1/users/{user_id}/preferences
Content-Type: application/json

Request:
{
  "channels": {
    "email": {
      "categories": {
        "marketing": false
      }
    }
  },
  "quiet_hours": {
    "enabled": true,
    "start": "23:00",
    "end": "07:00"
  }
}

Response: 200 OK
{
  "user_id": "user_123",
  "updated_at": "2024-01-15T10:35:00Z"
}
```

### Unsubscribe User

```http
POST /v1/users/{user_id}/unsubscribe
Content-Type: application/json

Request:
{
  "channel": "email",
  "category": "marketing",
  "reason": "too_frequent"
}

Response: 200 OK
{
  "user_id": "user_123",
  "channel": "email",
  "category": "marketing",
  "unsubscribed_at": "2024-01-15T10:30:00Z"
}
```

---

## 6. Notification Status API

### Get Notification Status

```http
GET /v1/notifications/{notification_id}
Authorization: Bearer <api_key>

Response: 200 OK
{
  "notification_id": "notif_abc123",
  "user_id": "user_123",
  "template_id": "order_confirmation",
  "status": "delivered",
  "channels": {
    "push": {
      "status": "delivered",
      "devices": [
        {
          "device_id": "dev_001",
          "platform": "ios",
          "status": "delivered",
          "delivered_at": "2024-01-15T10:30:05Z"
        },
        {
          "device_id": "dev_002",
          "platform": "android",
          "status": "delivered",
          "delivered_at": "2024-01-15T10:30:03Z"
        }
      ]
    },
    "email": {
      "status": "opened",
      "email": "user@example.com",
      "delivered_at": "2024-01-15T10:30:10Z",
      "opened_at": "2024-01-15T10:45:00Z",
      "clicked_at": "2024-01-15T10:45:30Z"
    }
  },
  "created_at": "2024-01-15T10:30:00Z"
}
```

### List User Notifications

```http
GET /v1/users/{user_id}/notifications
Authorization: Bearer <api_key>

Query Parameters:
- status: string (queued, sent, delivered, failed)
- channel: string
- since: ISO8601 timestamp
- limit: integer
- cursor: string

Response: 200 OK
{
  "notifications": [
    {
      "notification_id": "notif_abc123",
      "template_id": "order_confirmation",
      "title": "Order Confirmed!",
      "status": "delivered",
      "created_at": "2024-01-15T10:30:00Z"
    }
  ],
  "cursor": "...",
  "has_more": true
}
```

---

## 7. Analytics API

### Get Delivery Stats

```http
GET /v1/analytics/delivery
Authorization: Bearer <api_key>

Query Parameters:
- start_date: ISO8601
- end_date: ISO8601
- channel: string
- template_id: string
- granularity: string (hour, day, week)

Response: 200 OK
{
  "period": {
    "start": "2024-01-01T00:00:00Z",
    "end": "2024-01-15T00:00:00Z"
  },
  "totals": {
    "sent": 150000000,
    "delivered": 145000000,
    "failed": 5000000,
    "opened": 30000000,
    "clicked": 5000000
  },
  "by_channel": {
    "push": {
      "sent": 90000000,
      "delivered": 85500000,
      "delivery_rate": 0.95
    },
    "email": {
      "sent": 37500000,
      "delivered": 36750000,
      "delivery_rate": 0.98,
      "open_rate": 0.22,
      "click_rate": 0.03
    },
    "sms": {
      "sent": 22500000,
      "delivered": 22275000,
      "delivery_rate": 0.99
    }
  },
  "time_series": [
    {
      "timestamp": "2024-01-01T00:00:00Z",
      "sent": 10000000,
      "delivered": 9700000
    }
  ]
}
```

---

## 8. Database Schema

### Core Tables

```sql
-- Users table (reference)
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255),
    phone VARCHAR(20),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Device tokens table (sharded by user_id)
CREATE TABLE device_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    token TEXT NOT NULL,
    token_hash VARCHAR(64) NOT NULL,  -- For deduplication
    platform VARCHAR(20) NOT NULL,  -- ios, android, web
    app_version VARCHAR(20),
    device_model VARCHAR(100),
    os_version VARCHAR(50),
    timezone VARCHAR(50),
    language VARCHAR(10),
    is_active BOOLEAN DEFAULT TRUE,
    last_used_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE (token_hash),
    INDEX idx_device_tokens_user (user_id),
    INDEX idx_device_tokens_platform (platform, is_active)
);

-- User preferences table
CREATE TABLE user_preferences (
    user_id UUID PRIMARY KEY,
    push_enabled BOOLEAN DEFAULT TRUE,
    email_enabled BOOLEAN DEFAULT TRUE,
    sms_enabled BOOLEAN DEFAULT TRUE,
    channel_preferences JSONB DEFAULT '{}',
    quiet_hours_enabled BOOLEAN DEFAULT FALSE,
    quiet_hours_start TIME,
    quiet_hours_end TIME,
    timezone VARCHAR(50) DEFAULT 'UTC',
    topic_subscriptions TEXT[] DEFAULT '{}',
    unsubscribed_categories TEXT[] DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Templates table
CREATE TABLE notification_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    version INTEGER DEFAULT 1,
    push_config JSONB,
    email_config JSONB,
    sms_config JSONB,
    inapp_config JSONB,
    default_data JSONB DEFAULT '{}',
    category VARCHAR(50),
    tags TEXT[],
    is_active BOOLEAN DEFAULT TRUE,
    created_by UUID,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    INDEX idx_templates_category (category),
    INDEX idx_templates_name (name)
);

-- Template versions (for history)
CREATE TABLE template_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id UUID NOT NULL REFERENCES notification_templates(id),
    version INTEGER NOT NULL,
    config JSONB NOT NULL,
    created_by UUID,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE (template_id, version)
);
```

### Notification Tables

```sql
-- Notifications table (partitioned by created_at)
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    template_id UUID REFERENCES notification_templates(id),
    channels TEXT[] NOT NULL,
    data JSONB,
    priority VARCHAR(10) DEFAULT 'normal',
    status VARCHAR(20) DEFAULT 'pending',
    scheduled_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    processed_at TIMESTAMP WITH TIME ZONE,
    
    INDEX idx_notifications_user (user_id, created_at DESC),
    INDEX idx_notifications_status (status, created_at),
    INDEX idx_notifications_scheduled (scheduled_at) WHERE scheduled_at IS NOT NULL
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE notifications_2024_01 PARTITION OF notifications
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Channel delivery status
CREATE TABLE notification_deliveries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notification_id UUID NOT NULL,
    channel VARCHAR(20) NOT NULL,
    device_id UUID,  -- For push
    recipient VARCHAR(255),  -- Email or phone
    status VARCHAR(20) DEFAULT 'pending',
    provider VARCHAR(50),
    provider_message_id VARCHAR(255),
    error_code VARCHAR(50),
    error_message TEXT,
    attempts INTEGER DEFAULT 0,
    sent_at TIMESTAMP WITH TIME ZONE,
    delivered_at TIMESTAMP WITH TIME ZONE,
    opened_at TIMESTAMP WITH TIME ZONE,
    clicked_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    INDEX idx_deliveries_notification (notification_id),
    INDEX idx_deliveries_status (channel, status),
    INDEX idx_deliveries_provider (provider, provider_message_id)
);
```

### Analytics Tables (ClickHouse)

```sql
-- Notification events (ClickHouse)
CREATE TABLE notification_events (
    event_id UUID,
    notification_id UUID,
    user_id UUID,
    template_id UUID,
    channel String,
    event_type String,  -- sent, delivered, opened, clicked, failed
    device_id Nullable(UUID),
    provider String,
    error_code Nullable(String),
    timestamp DateTime,
    date Date MATERIALIZED toDate(timestamp)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, channel, event_type, timestamp);

-- Aggregated stats (materialized view)
CREATE MATERIALIZED VIEW notification_stats_daily
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, channel, template_id)
AS SELECT
    toDate(timestamp) as date,
    channel,
    template_id,
    countIf(event_type = 'sent') as sent,
    countIf(event_type = 'delivered') as delivered,
    countIf(event_type = 'opened') as opened,
    countIf(event_type = 'clicked') as clicked,
    countIf(event_type = 'failed') as failed
FROM notification_events
GROUP BY date, channel, template_id;
```

---

## 9. Webhook Events

### Webhook Configuration

```http
POST /v1/webhooks
Content-Type: application/json

Request:
{
  "url": "https://myapp.com/webhooks/notifications",
  "events": ["delivered", "opened", "clicked", "failed", "bounced"],
  "secret": "webhook_secret_for_signature"
}

Response: 201 Created
{
  "id": "webhook_abc123",
  "url": "https://myapp.com/webhooks/notifications",
  "events": ["delivered", "opened", "clicked", "failed", "bounced"],
  "status": "active",
  "created_at": "2024-01-15T10:30:00Z"
}
```

### Webhook Payload

```json
// Delivery event
{
  "event_id": "evt_abc123",
  "event_type": "delivered",
  "timestamp": "2024-01-15T10:30:00Z",
  "notification_id": "notif_xyz789",
  "user_id": "user_123",
  "channel": "push",
  "device_id": "dev_001",
  "data": {
    "provider": "fcm",
    "provider_message_id": "fcm_msg_123"
  }
}

// Bounce event
{
  "event_id": "evt_def456",
  "event_type": "bounced",
  "timestamp": "2024-01-15T10:30:00Z",
  "notification_id": "notif_xyz789",
  "user_id": "user_123",
  "channel": "email",
  "data": {
    "bounce_type": "hard",
    "bounce_reason": "invalid_email",
    "email": "user@invalid.com"
  }
}
```

---

## 10. Error Responses

### Standard Error Format

```json
{
  "error": {
    "code": "invalid_template",
    "message": "Template 'xyz' not found or inactive",
    "details": {
      "template_id": "xyz"
    },
    "request_id": "req_abc123"
  }
}
```

### Error Codes

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | invalid_request | Malformed request |
| 400 | invalid_template | Template not found |
| 400 | invalid_channel | Unsupported channel |
| 400 | missing_user_id | User ID required |
| 401 | unauthorized | Invalid API key |
| 403 | forbidden | Insufficient permissions |
| 404 | user_not_found | User doesn't exist |
| 409 | duplicate_request | Idempotency key reused |
| 429 | rate_limited | Too many requests |
| 500 | internal_error | Server error |
| 503 | provider_unavailable | External provider down |

---

## 11. Idempotency Model

### What is Idempotency?

An operation is **idempotent** if performing it multiple times has the same effect as performing it once. This is critical for notification systems to prevent duplicate notifications when network retries occur.

### Idempotent Operations in Notification System

| Operation | Idempotent? | Mechanism |
|-----------|-------------|-----------|
| **Send Notification** | ✅ Yes | Idempotency key prevents duplicate notifications |
| **Update Preferences** | ✅ Yes | Idempotent state update |
| **Register Device** | ✅ Yes | Device token uniqueness |
| **Unregister Device** | ✅ Yes | DELETE is idempotent (safe to retry) |
| **Deliver Notification** | ⚠️ At-least-once | Provider-level deduplication (collapse_key) |

### Idempotency Implementation

**1. Send Notification with Idempotency Key:**

```java
@PostMapping("/v1/notifications")
public ResponseEntity<NotificationResponse> sendNotification(
        @RequestBody SendNotificationRequest request,
        @RequestHeader("X-Idempotency-Key") String idempotencyKey) {
    
    // Check if notification already sent
    Notification existing = notificationRepository.findByIdempotencyKey(idempotencyKey);
    if (existing != null) {
        return ResponseEntity.ok(NotificationResponse.from(existing));
    }
    
    // Create notification record
    String notificationId = generateNotificationId();
    Notification notification = Notification.builder()
        .id(notificationId)
        .idempotencyKey(idempotencyKey)
        .userId(request.getUserId())
        .templateId(request.getTemplateId())
        .channels(request.getChannels())
        .data(request.getData())
        .options(request.getOptions())
        .status(NotificationStatus.QUEUED)
        .createdAt(Instant.now())
        .build();
    
    notification = notificationRepository.save(notification);
    
    // Publish to Kafka for async processing (idempotent via notification_id)
    kafkaTemplate.send("notification-requests", 
        request.getUserId(), 
        NotificationEvent.from(notification)
    );
    
    return ResponseEntity.status(202).body(NotificationResponse.from(notification));
}
```

**2. Provider-Level Deduplication (Push Notifications):**

```java
// Use collapse_key for FCM/APNS to prevent duplicate notifications
public void sendPushNotification(Notification notification, String deviceToken) {
    // Check if same notification already sent recently
    String dedupKey = String.format("push:%s:%s:%s",
        notification.getUserId(),
        notification.getTemplateId(),
        notification.getOptions().getCollapseKey()
    );
    
    Boolean isNew = redisTemplate.opsForValue()
        .setIfAbsent(dedupKey, "1", Duration.ofHours(1));
    
    if (Boolean.TRUE.equals(isNew)) {
        // New notification, send it
        PushMessage message = PushMessage.builder()
            .token(deviceToken)
            .title(notification.getTitle())
            .body(notification.getBody())
            .data(notification.getData())
            .collapseKey(notification.getOptions().getCollapseKey()) // Deduplication
            .priority(notification.getOptions().getPriority())
            .ttl(notification.getOptions().getTtl())
            .build();
        
        pushProvider.send(message);
    }
    // Duplicate notification, ignore
}
```

**3. Email Deduplication:**

```java
// Deduplicate emails by (user_id, template_id, data_hash)
public void sendEmail(Notification notification) {
    String dataHash = DigestUtils.sha256Hex(
        notification.getTemplateId() + 
        notification.getData().toString()
    );
    
    String dedupKey = String.format("email:%s:%s:%s",
        notification.getUserId(),
        notification.getTemplateId(),
        dataHash
    );
    
    Boolean isNew = redisTemplate.opsForValue()
        .setIfAbsent(dedupKey, "1", Duration.ofHours(24));
    
    if (Boolean.TRUE.equals(isNew)) {
        // New email, send it
        emailService.send(notification);
    }
    // Duplicate email, ignore
}
```

**4. SMS Deduplication:**

```java
// Deduplicate SMS by (user_id, template_id, timestamp)
public void sendSMS(Notification notification) {
    long timestamp = System.currentTimeMillis() / 1000; // Round to second
    String dedupKey = String.format("sms:%s:%s:%d",
        notification.getUserId(),
        notification.getTemplateId(),
        timestamp
    );
    
    Boolean isNew = redisTemplate.opsForValue()
        .setIfAbsent(dedupKey, "1", Duration.ofMinutes(5));
    
    if (Boolean.TRUE.equals(isNew)) {
        // New SMS, send it
        smsService.send(notification);
    }
    // Duplicate SMS, ignore
}
```

### Idempotency Key Generation

**Client-Side (Required):**
```javascript
// Generate UUID v4 for each notification request
const idempotencyKey = crypto.randomUUID();

fetch('/v1/notifications', {
    method: 'POST',
    headers: {
        'Authorization': `Bearer ${apiKey}`,
        'X-Idempotency-Key': idempotencyKey
    },
    body: notificationData
});
```

**Server-Side (Fallback):**
```java
// If client doesn't provide key, generate from request hash
String idempotencyKey = DigestUtils.sha256Hex(
    request.getUserId() + 
    request.getTemplateId() +
    request.getData().toString() +
    Instant.now().toString()
);
```

### Duplicate Detection Window

| Operation | Deduplication Window | Storage |
|-----------|---------------------|---------|
| Send Notification | 24 hours | PostgreSQL (idempotency_key unique index) |
| Push Notifications | 1 hour | Redis + Provider collapse_key |
| Email | 24 hours | Redis |
| SMS | 5 minutes | Redis |
| Device Registration | Forever | PostgreSQL (device_token unique) |

### Idempotency Key Storage

```sql
-- Add idempotency_key to notifications
ALTER TABLE notifications ADD COLUMN idempotency_key VARCHAR(255);
CREATE UNIQUE INDEX idx_notifications_idempotency_key 
ON notifications(idempotency_key) WHERE idempotency_key IS NOT NULL;

-- Device tokens are naturally unique
CREATE UNIQUE INDEX idx_device_tokens_token ON device_tokens(token);
```

