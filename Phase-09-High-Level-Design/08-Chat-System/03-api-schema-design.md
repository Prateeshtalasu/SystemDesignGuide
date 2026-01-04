# Chat System - API & Schema Design

## API Design Philosophy

Chat APIs prioritize:

1. **Real-time**: WebSocket for instant message delivery
2. **Reliability**: Guaranteed delivery with acknowledgments
3. **Ordering**: Messages must arrive in sequence
4. **Efficiency**: Minimize bandwidth for mobile

---

## Base URL Structure

```
REST API:     https://api.chat.com/v1
WebSocket:    wss://ws.chat.com/v1
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

## Authentication

### JWT Token Authentication

```http
GET /v1/conversations
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

---

## Rate Limiting Headers

Every response includes rate limit information:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
```

**Rate Limits:**

| Tier | Requests/minute |
|------|-----------------|
| Free | 100 |
| Pro | 1000 |
| Enterprise | 10000 |

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
| 400 | INVALID_MESSAGE | Message content is invalid |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Conversation or message not found |
| 409 | CONFLICT | Message conflict (duplicate) |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

**Error Response Examples:**

```json
// 400 Bad Request - Invalid message
{
  "error": {
    "code": "INVALID_MESSAGE",
    "message": "Message content is invalid",
    "details": {
      "field": "content",
      "reason": "Message cannot be empty"
    },
    "request_id": "req_abc123"
  }
}

// 404 Not Found - Conversation not found
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Conversation not found",
    "details": {
      "field": "conversation_id",
      "reason": "Conversation 'conv_abc123' does not exist"
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
| POST /v1/messages | Yes | Idempotency-Key header + client_message_id |
| PUT /v1/messages/{id} | Yes | Idempotency-Key or version-based |
| DELETE /v1/messages/{id} | Yes | Safe to retry (idempotent by design) |
| POST /v1/conversations | Yes | Idempotency-Key header |

**Implementation Example:**

```java
@PostMapping("/v1/messages")
public ResponseEntity<MessageResponse> sendMessage(
        @RequestBody SendMessageRequest request,
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {
    
    // Use client_message_id as idempotency key if not provided
    String key = idempotencyKey != null ? idempotencyKey : request.getClientMessageId();
    
    // Check for existing idempotency key
    if (key != null) {
        String cacheKey = "idempotency:" + key;
        String cachedResponse = redisTemplate.opsForValue().get(cacheKey);
        if (cachedResponse != null) {
            MessageResponse response = objectMapper.readValue(cachedResponse, MessageResponse.class);
            return ResponseEntity.status(response.getStatus()).body(response);
        }
    }
    
    // Check for duplicate client_message_id in conversation
    if (request.getClientMessageId() != null) {
        Message existing = messageRepository.findByConversationIdAndClientMessageId(
            request.getConversationId(), 
            request.getClientMessageId()
        );
        if (existing != null) {
            return ResponseEntity.ok(MessageResponse.from(existing));
        }
    }
    
    // Create message
    Message message = messageService.createMessage(request, key);
    MessageResponse response = MessageResponse.from(message);
    
    // Cache response if key provided
    if (key != null) {
        String cacheKey = "idempotency:" + key;
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

## Core API Endpoints

### 1. Send Message

**Endpoint:** `POST /v1/messages`

**Purpose:** Send a message to a conversation

**Request:**

```http
POST /v1/messages HTTP/1.1
Host: api.chat.com
Authorization: Bearer user_token
Content-Type: application/json
X-Request-ID: req_123

{
  "conversation_id": "conv_abc123",
  "content": {
    "type": "text",
    "text": "Hello, how are you?"
  },
  "client_message_id": "client_msg_456",
  "reply_to": null
}
```

**Response (202 Accepted):**

```json
{
  "message_id": "msg_xyz789",
  "client_message_id": "client_msg_456",
  "conversation_id": "conv_abc123",
  "status": "sent",
  "timestamp": "2024-01-20T15:30:00.123Z",
  "sequence_number": 1542
}
```

**Message Content Types:**

```json
// Text
{"type": "text", "text": "Hello!"}

// Image
{"type": "image", "media_id": "media_123", "thumbnail_url": "...", "width": 1080, "height": 1920}

// Video
{"type": "video", "media_id": "media_456", "thumbnail_url": "...", "duration": 30}

// Document
{"type": "document", "media_id": "media_789", "filename": "report.pdf", "size": 1024000}

// Location
{"type": "location", "latitude": 40.7128, "longitude": -74.0060, "name": "New York"}
```

---

### 2. Get Messages (Sync)

**Endpoint:** `GET /v1/conversations/{conversation_id}/messages`

**Purpose:** Fetch message history for a conversation

**Request:**

```http
GET /v1/conversations/conv_abc123/messages?before=msg_xyz&limit=50 HTTP/1.1
Host: api.chat.com
Authorization: Bearer user_token
```

**Query Parameters:**

| Parameter | Type   | Required | Description                    |
| --------- | ------ | -------- | ------------------------------ |
| before    | string | No       | Fetch messages before this ID  |
| after     | string | No       | Fetch messages after this ID   |
| limit     | int    | No       | Max messages (default 50)      |

**Response:**

```json
{
  "messages": [
    {
      "message_id": "msg_xyz789",
      "sender_id": "user_alice",
      "content": {
        "type": "text",
        "text": "Hello, how are you?"
      },
      "timestamp": "2024-01-20T15:30:00.123Z",
      "sequence_number": 1542,
      "status": "read",
      "reactions": [
        {"emoji": "👍", "user_ids": ["user_bob"]}
      ]
    }
  ],
  "has_more": true,
  "oldest_message_id": "msg_abc123"
}
```

---

### 3. Create Conversation

**Endpoint:** `POST /v1/conversations`

**Request:**

```http
POST /v1/conversations HTTP/1.1
Host: api.chat.com
Authorization: Bearer user_token
Content-Type: application/json

{
  "type": "group",
  "name": "Family Chat",
  "participants": ["user_bob", "user_charlie", "user_diana"],
  "avatar_url": "https://cdn.chat.com/avatars/family.jpg"
}
```

**Response (201 Created):**

```json
{
  "conversation_id": "conv_new123",
  "type": "group",
  "name": "Family Chat",
  "participants": [
    {"user_id": "user_alice", "role": "admin"},
    {"user_id": "user_bob", "role": "member"},
    {"user_id": "user_charlie", "role": "member"},
    {"user_id": "user_diana", "role": "member"}
  ],
  "created_at": "2024-01-20T15:30:00Z"
}
```

---

### 4. Update Message Status

**Endpoint:** `POST /v1/messages/status`

**Purpose:** Mark messages as delivered/read

**Request:**

```http
POST /v1/messages/status HTTP/1.1
Host: api.chat.com
Authorization: Bearer user_token
Content-Type: application/json

{
  "conversation_id": "conv_abc123",
  "status": "read",
  "up_to_message_id": "msg_xyz789"
}
```

**Response:**

```json
{
  "updated_count": 15,
  "conversation_id": "conv_abc123"
}
```

---

### 5. WebSocket Protocol

**Connection:**

```
wss://ws.chat.com/connect?token=user_token&device_id=device_123
```

**Client → Server Messages:**

```json
// Send message
{
  "type": "message",
  "id": "client_req_123",
  "payload": {
    "conversation_id": "conv_abc",
    "content": {"type": "text", "text": "Hello!"}
  }
}

// Acknowledge receipt
{
  "type": "ack",
  "message_id": "msg_xyz789"
}

// Typing indicator
{
  "type": "typing",
  "conversation_id": "conv_abc",
  "is_typing": true
}

// Presence update
{
  "type": "presence",
  "status": "online"
}

// Heartbeat
{
  "type": "ping"
}
```

**Server → Client Messages:**

```json
// New message
{
  "type": "message",
  "payload": {
    "message_id": "msg_xyz789",
    "conversation_id": "conv_abc",
    "sender_id": "user_bob",
    "content": {"type": "text", "text": "Hi there!"},
    "timestamp": "2024-01-20T15:30:00.123Z",
    "sequence_number": 1543
  }
}

// Message status update
{
  "type": "status_update",
  "payload": {
    "message_id": "msg_xyz789",
    "status": "delivered",
    "user_id": "user_bob"
  }
}

// Typing indicator
{
  "type": "typing",
  "payload": {
    "conversation_id": "conv_abc",
    "user_id": "user_bob",
    "is_typing": true
  }
}

// Presence update
{
  "type": "presence",
  "payload": {
    "user_id": "user_bob",
    "status": "online",
    "last_seen": null
  }
}

// Heartbeat response
{
  "type": "pong"
}
```

---

## Entity Relationship Diagram

```
┌─────────────────────┐       ┌─────────────────────┐
│       users         │       │   conversations     │
├─────────────────────┤       ├─────────────────────┤
│ id (PK)             │       │ id (PK)             │
│ user_id (unique)    │       │ conversation_id     │
│ username            │       │ type                │
│ phone_number        │       │ name                │
│ public_key          │       │ last_message_at     │
└─────────────────────┘       └─────────────────────┘
         │                             │
         │                             │
         └──────────┬──────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────┐
│            conversation_participants                 │
├─────────────────────────────────────────────────────┤
│ id (PK)                                             │
│ conversation_id (FK)                                │
│ user_id (FK)                                        │
│ role                                                │
│ last_read_message_id                                │
│ muted_until                                         │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  messages (Cassandra)                │
├─────────────────────────────────────────────────────┤
│ conversation_id (PK)                                │
│ message_id (CK)                                     │
│ sender_id                                           │
│ content_type                                        │
│ content                                             │
│ sequence_number                                     │
│ created_at                                          │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  REDIS CACHE                         │
├─────────────────────────────────────────────────────┤
│ session:{user_id}:{device_id} → connection info     │
│ presence:{user_id} → online/offline                 │
│ pending:{user_id} → undelivered messages            │
│ typing:{conv_id}:{user_id} → typing indicator       │
└─────────────────────────────────────────────────────┘
```

---

## Message Delivery Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          MESSAGE DELIVERY PROTOCOL                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘

Sender                    Server                    Recipient
  │                         │                          │
  │ 1. Send message         │                          │
  │ ───────────────────────>│                          │
  │                         │                          │
  │                         │ 2. Store message         │
  │                         │ ───────────────>         │
  │                         │                          │
  │ 3. ACK (sent)           │                          │
  │ <───────────────────────│                          │
  │                         │                          │
  │                         │ 4. Push to recipient     │
  │                         │ ─────────────────────────>
  │                         │                          │
  │                         │ 5. ACK (received)        │
  │                         │ <─────────────────────────
  │                         │                          │
  │ 6. Status: delivered    │                          │
  │ <───────────────────────│                          │
  │                         │                          │
  │                         │ 7. User reads message    │
  │                         │                          │
  │                         │ 8. Read receipt          │
  │                         │ <─────────────────────────
  │                         │                          │
  │ 9. Status: read         │                          │
  │ <───────────────────────│                          │
```

---

## Idempotency Model

### What is Idempotency?

An operation is **idempotent** if performing it multiple times has the same effect as performing it once. This is critical for chat systems to prevent duplicate messages when network retries occur.

### Idempotent Operations in Chat System

| Operation | Idempotent? | Mechanism |
|-----------|-------------|-----------|
| **Send Message** | ✅ Yes | Client message ID prevents duplicates |
| **Mark as Read** | ✅ Yes | Idempotent state update |
| **Delete Message** | ✅ Yes | DELETE is idempotent (safe to retry) |
| **Update Conversation** | ✅ Yes | Version-based updates prevent conflicts |
| **Deliver Message** | ⚠️ At-least-once | Client deduplication by message_id |

### Idempotency Implementation

**1. Message Sending with Client Message ID:**

```java
@PostMapping("/v1/messages")
public ResponseEntity<MessageResponse> sendMessage(
        @RequestBody SendMessageRequest request,
        @RequestHeader("X-Request-ID") String requestId) {
    
    // Check for duplicate using client_message_id
    if (request.getClientMessageId() != null) {
        Message existing = messageRepository.findByClientMessageId(
            request.getConversationId(),
            request.getClientMessageId(),
            request.getSenderId()
        );
        
        if (existing != null) {
            // Duplicate request, return existing message
            return ResponseEntity.ok(MessageResponse.from(existing));
        }
    }
    
    // Generate server-side message ID
    String messageId = generateMessageId();
    
    // Assign sequence number (per conversation)
    long sequenceNumber = sequenceService.getNextSequence(
        request.getConversationId()
    );
    
    // Create message
    Message message = Message.builder()
        .id(messageId)
        .clientMessageId(request.getClientMessageId())
        .conversationId(request.getConversationId())
        .senderId(request.getSenderId())
        .content(request.getContent())
        .sequenceNumber(sequenceNumber)
        .status(MessageStatus.SENT)
        .createdAt(Instant.now())
        .build();
    
    // Store message (idempotent via client_message_id unique constraint)
    message = messageRepository.save(message);
    
    // Publish to Kafka for async delivery (idempotent via message_id)
    kafkaTemplate.send("message-events", 
        request.getRecipientId(), 
        MessageEvent.from(message)
    );
    
    return ResponseEntity.status(202).body(MessageResponse.from(message));
}
```

**2. Client-Side Deduplication:**

```java
// Client maintains a set of received message IDs
public class MessageDeduplicator {
    private final Set<String> receivedMessageIds = new ConcurrentHashMap<>().newKeySet();
    private static final int MAX_DEDUP_SIZE = 10000;
    
    public boolean isDuplicate(String messageId) {
        if (receivedMessageIds.contains(messageId)) {
            return true; // Duplicate
        }
        
        receivedMessageIds.add(messageId);
        
        // Evict oldest if set too large
        if (receivedMessageIds.size() > MAX_DEDUP_SIZE) {
            // Remove oldest 1000 entries (simple eviction)
            Iterator<String> it = receivedMessageIds.iterator();
            for (int i = 0; i < 1000 && it.hasNext(); i++) {
                it.next();
                it.remove();
            }
        }
        
        return false; // New message
    }
}
```

**3. Read Receipt Idempotency:**

```java
@PostMapping("/v1/messages/{messageId}/read")
public ResponseEntity<Void> markAsRead(
        @PathVariable String messageId,
        @RequestParam String userId) {
    
    // Check if already marked as read
    ReadReceipt existing = readReceiptRepository.findByMessageIdAndUserId(
        messageId, userId
    );
    
    if (existing != null) {
        // Already read, idempotent
        return ResponseEntity.ok().build();
    }
    
    // Create read receipt
    ReadReceipt receipt = ReadReceipt.builder()
        .messageId(messageId)
        .userId(userId)
        .readAt(Instant.now())
        .build();
    
    readReceiptRepository.save(receipt);
    
    // Update message status (idempotent)
    messageRepository.updateReadStatus(messageId, userId);
    
    return ResponseEntity.ok().build();
}
```

**4. Message Delivery Deduplication (Server-Side):**

```java
// Deduplicate message delivery attempts
public void deliverMessage(Message message, String recipientId, String deviceId) {
    String dedupKey = String.format("delivery:%s:%s:%s", 
        message.getId(), recipientId, deviceId);
    
    // Set if absent (idempotent)
    Boolean isNew = redisTemplate.opsForValue()
        .setIfAbsent(dedupKey, "1", Duration.ofMinutes(5));
    
    if (Boolean.TRUE.equals(isNew)) {
        // New delivery attempt
        deliverToDevice(message, recipientId, deviceId);
    }
    // Duplicate delivery attempt, ignore
}
```

### Client Message ID Generation

**Client-Side (Required):**
```javascript
// Generate UUID v4 for each message
const clientMessageId = crypto.randomUUID();

// Send message with client ID
fetch('/v1/messages', {
    method: 'POST',
    headers: {
        'Authorization': `Bearer ${token}`,
        'X-Request-ID': crypto.randomUUID()
    },
    body: JSON.stringify({
        conversation_id: 'conv_123',
        content: { type: 'text', text: 'Hello!' },
        client_message_id: clientMessageId
    })
});
```

### Duplicate Detection Window

| Operation | Deduplication Window | Storage |
|-----------|---------------------|---------|
| Message Sending | 24 hours | Cassandra (client_message_id unique) |
| Message Delivery | 5 minutes | Redis |
| Read Receipts | Forever | PostgreSQL (unique constraint) |
| Typing Indicators | 30 seconds | Redis (TTL) |

### Idempotency Key Storage

```sql
-- Add client_message_id with unique constraint
ALTER TABLE messages ADD COLUMN client_message_id VARCHAR(255);
CREATE UNIQUE INDEX idx_messages_client_id ON messages(
    conversation_id, client_message_id, sender_id
) WHERE client_message_id IS NOT NULL;

-- Read receipts are naturally idempotent (unique constraint)
CREATE UNIQUE INDEX idx_read_receipts_unique ON read_receipts(
    message_id, user_id
);
```

---

## Summary

| Component           | Technology/Approach                     |
| ------------------- | --------------------------------------- |
| Real-time protocol  | WebSocket with binary framing           |
| Message storage     | Cassandra (partitioned by conversation) |
| User/Conversation   | PostgreSQL                              |
| Session/Presence    | Redis with TTL                          |
| Message queue       | Kafka for async processing              |
| Delivery guarantee  | At-least-once with client dedup         |
| Ordering            | Sequence numbers per conversation       |

