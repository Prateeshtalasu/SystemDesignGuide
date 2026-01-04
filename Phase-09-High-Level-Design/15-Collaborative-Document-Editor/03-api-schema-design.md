# Collaborative Document Editor - API & Schema Design

## 1. API Design Principles

### RESTful API for Metadata

```
Base URL: https://api.docs.example.com/v1

Authentication: OAuth 2.0 / JWT Bearer Token
Rate Limiting: 1000 requests/minute
Versioning: URL path versioning (/v1/, /v2/)
Content-Type: application/json
```

### WebSocket API for Real-time

```
WebSocket URL: wss://realtime.docs.example.com/v1/documents/{doc_id}

Protocol: Custom binary protocol over WebSocket
Heartbeat: Every 30 seconds
Reconnection: Exponential backoff with jitter
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
| 400 | INVALID_OPERATION | Document operation is invalid |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Document not found |
| 409 | CONFLICT | Document version conflict |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

**Error Response Examples:**

```json
// 400 Bad Request - Invalid operation
{
  "error": {
    "code": "INVALID_OPERATION",
    "message": "Document operation is invalid",
    "details": {
      "field": "operation",
      "reason": "Operation index out of bounds"
    },
    "request_id": "req_abc123"
  }
}

// 409 Conflict - Version conflict
{
  "error": {
    "code": "CONFLICT",
    "message": "Document version conflict",
    "details": {
      "field": "version",
      "reason": "Document has been modified. Please refresh and retry."
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
      "limit": 1000,
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
| POST /v1/documents | Yes | Idempotency-Key header |
| PUT /v1/documents/{id} | Yes | Idempotency-Key or version-based |
| DELETE /v1/documents/{id} | Yes | Safe to retry (idempotent by design) |
| POST /v1/documents/{id}/operations | Yes | Idempotency-Key header (for OT/CRDT operations) |

**Implementation Example:**

```java
@PostMapping("/v1/documents")
public ResponseEntity<DocumentResponse> createDocument(
        @RequestBody CreateDocumentRequest request,
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {
    
    // Check for existing idempotency key
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        String cachedResponse = redisTemplate.opsForValue().get(cacheKey);
        if (cachedResponse != null) {
            DocumentResponse response = objectMapper.readValue(cachedResponse, DocumentResponse.class);
            return ResponseEntity.status(response.getStatus()).body(response);
        }
    }
    
    // Create document
    Document document = documentService.createDocument(request, idempotencyKey);
    DocumentResponse response = DocumentResponse.from(document);
    
    // Cache response if idempotency key provided
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        redisTemplate.opsForValue().set(
            cacheKey, 
            objectMapper.writeValueAsString(response),
            Duration.ofHours(24)
        );
    }
    
    return ResponseEntity.status(201).body(response);
}
```

---

## 2. Document Operations API

### Create Document

```http
POST /v1/documents
Content-Type: application/json
Authorization: Bearer <token>

Request:
{
  "title": "Project Proposal",
  "template_id": "blank",  // or specific template
  "folder_id": "folder_123",
  "initial_content": {
    "type": "doc",
    "content": []
  }
}

Response: 201 Created
{
  "id": "doc_abc123",
  "title": "Project Proposal",
  "owner": {
    "id": "user_123",
    "email": "user@example.com",
    "name": "John Doe"
  },
  "created_at": "2024-01-15T10:30:00Z",
  "modified_at": "2024-01-15T10:30:00Z",
  "version": 1,
  "permissions": {
    "role": "owner",
    "can_edit": true,
    "can_comment": true,
    "can_share": true
  },
  "url": "https://docs.example.com/d/doc_abc123"
}
```

### Get Document

```http
GET /v1/documents/{doc_id}
Authorization: Bearer <token>

Query Parameters:
- include_content: boolean (default: false)
- version: integer (specific version, default: latest)

Response: 200 OK
{
  "id": "doc_abc123",
  "title": "Project Proposal",
  "owner": {
    "id": "user_123",
    "email": "user@example.com",
    "name": "John Doe"
  },
  "created_at": "2024-01-15T10:30:00Z",
  "modified_at": "2024-01-15T14:30:00Z",
  "version": 42,
  "word_count": 1500,
  "character_count": 8500,
  "collaborators": [
    {
      "id": "user_456",
      "name": "Jane Smith",
      "avatar_url": "https://...",
      "last_viewed": "2024-01-15T14:25:00Z"
    }
  ],
  "permissions": {
    "role": "editor",
    "can_edit": true,
    "can_comment": true,
    "can_share": false
  },
  "content": {
    // Only if include_content=true
    "type": "doc",
    "content": [...]
  }
}
```

### Update Document Metadata

```http
PATCH /v1/documents/{doc_id}
Content-Type: application/json

Request:
{
  "title": "Updated Project Proposal",
  "folder_id": "folder_456"
}

Response: 200 OK
{
  "id": "doc_abc123",
  "title": "Updated Project Proposal",
  // ... updated document
}
```

### Delete Document

```http
DELETE /v1/documents/{doc_id}
Authorization: Bearer <token>

Query Parameters:
- permanent: boolean (default: false, moves to trash)

Response: 204 No Content
```

---

## 3. Real-time Collaboration Protocol

### WebSocket Connection

```javascript
// Client connects to document
const ws = new WebSocket('wss://realtime.docs.example.com/v1/documents/doc_abc123');

// Authentication message
ws.send(JSON.stringify({
  type: 'auth',
  token: 'jwt_token_here',
  client_id: 'client_uuid',
  last_version: 41  // Last known version for sync
}));

// Server response
{
  type: 'auth_success',
  user_id: 'user_123',
  document_version: 42,
  missing_operations: [...],  // Ops since version 41
  active_users: [
    { user_id: 'user_456', cursor: { line: 10, ch: 5 }, color: '#FF5733' }
  ]
}
```

### Operation Message Format

```javascript
// Client sends operation
{
  type: 'operation',
  client_id: 'client_uuid',
  parent_version: 42,
  operation: {
    type: 'insert',
    position: 150,
    content: 'Hello ',
    attributes: { bold: true }
  },
  timestamp: 1705320600000
}

// Server broadcasts to all clients
{
  type: 'operation',
  user_id: 'user_123',
  version: 43,
  operation: {
    type: 'insert',
    position: 150,
    content: 'Hello ',
    attributes: { bold: true }
  },
  timestamp: 1705320600000
}

// Acknowledgment to sender
{
  type: 'ack',
  client_id: 'client_uuid',
  version: 43
}
```

### Operation Types

```javascript
// Insert operation
{
  type: 'insert',
  position: 150,
  content: 'Hello World',
  attributes: {
    bold: true,
    italic: false,
    font_size: 12,
    font_family: 'Arial'
  }
}

// Delete operation
{
  type: 'delete',
  position: 150,
  length: 11
}

// Format operation
{
  type: 'format',
  position: 150,
  length: 11,
  attributes: {
    bold: true
  }
}

// Retain operation (for OT)
{
  type: 'retain',
  length: 150
}
```

### Cursor/Presence Updates

```javascript
// Client sends cursor position
{
  type: 'cursor',
  client_id: 'client_uuid',
  position: {
    anchor: { line: 10, ch: 5 },
    head: { line: 10, ch: 15 }  // Selection end
  }
}

// Server broadcasts presence
{
  type: 'presence',
  users: [
    {
      user_id: 'user_123',
      name: 'John Doe',
      color: '#FF5733',
      cursor: { line: 10, ch: 5 },
      selection: { start: 150, end: 160 },
      last_active: 1705320600000
    },
    {
      user_id: 'user_456',
      name: 'Jane Smith',
      color: '#33FF57',
      cursor: { line: 25, ch: 0 },
      selection: null,
      last_active: 1705320595000
    }
  ]
}
```

---

## 4. Comments API

### Add Comment

```http
POST /v1/documents/{doc_id}/comments
Content-Type: application/json

Request:
{
  "content": "This section needs more detail.",
  "anchor": {
    "start": 150,
    "end": 200,
    "quoted_text": "project timeline"
  }
}

Response: 201 Created
{
  "id": "comment_xyz",
  "author": {
    "id": "user_123",
    "name": "John Doe",
    "avatar_url": "https://..."
  },
  "content": "This section needs more detail.",
  "anchor": {
    "start": 150,
    "end": 200,
    "quoted_text": "project timeline"
  },
  "created_at": "2024-01-15T10:30:00Z",
  "resolved": false,
  "replies": []
}
```

### Reply to Comment

```http
POST /v1/documents/{doc_id}/comments/{comment_id}/replies
Content-Type: application/json

Request:
{
  "content": "Good point, I'll add more information."
}

Response: 201 Created
{
  "id": "reply_abc",
  "author": {
    "id": "user_456",
    "name": "Jane Smith"
  },
  "content": "Good point, I'll add more information.",
  "created_at": "2024-01-15T10:35:00Z"
}
```

### List Comments

```http
GET /v1/documents/{doc_id}/comments
Authorization: Bearer <token>

Query Parameters:
- resolved: boolean (filter by resolved status)
- limit: integer
- cursor: string

Response: 200 OK
{
  "comments": [
    {
      "id": "comment_xyz",
      "author": {...},
      "content": "This section needs more detail.",
      "anchor": {...},
      "created_at": "2024-01-15T10:30:00Z",
      "resolved": false,
      "replies": [...]
    }
  ],
  "cursor": "eyJsYXN0X2lkIjoiY29tbWVudF94eXoifQ==",
  "has_more": false
}
```

---

## 5. Version History API

### List Versions

```http
GET /v1/documents/{doc_id}/versions
Authorization: Bearer <token>

Query Parameters:
- limit: integer (default: 20)
- cursor: string

Response: 200 OK
{
  "versions": [
    {
      "version": 42,
      "timestamp": "2024-01-15T14:30:00Z",
      "author": {
        "id": "user_123",
        "name": "John Doe"
      },
      "changes_summary": "Added introduction section",
      "word_count_delta": +150
    },
    {
      "version": 41,
      "timestamp": "2024-01-15T14:25:00Z",
      "author": {
        "id": "user_456",
        "name": "Jane Smith"
      },
      "changes_summary": "Fixed typos",
      "word_count_delta": -2
    }
  ],
  "cursor": "eyJ2ZXJzaW9uIjo0MH0=",
  "has_more": true
}
```

### Get Specific Version

```http
GET /v1/documents/{doc_id}/versions/{version}
Authorization: Bearer <token>

Response: 200 OK
{
  "version": 41,
  "timestamp": "2024-01-15T14:25:00Z",
  "author": {...},
  "content": {
    "type": "doc",
    "content": [...]
  }
}
```

### Restore Version

```http
POST /v1/documents/{doc_id}/versions/{version}/restore
Authorization: Bearer <token>

Response: 200 OK
{
  "id": "doc_abc123",
  "version": 43,  // New version created from restored
  // ... document object
}
```

### Name a Version

```http
POST /v1/documents/{doc_id}/versions/{version}/name
Content-Type: application/json

Request:
{
  "name": "Final Draft v1"
}

Response: 200 OK
{
  "version": 42,
  "name": "Final Draft v1",
  "timestamp": "2024-01-15T14:30:00Z"
}
```

---

## 6. Sharing API

### Share Document

```http
POST /v1/documents/{doc_id}/share
Content-Type: application/json

Request:
{
  "shares": [
    {
      "email": "colleague@example.com",
      "role": "editor",  // viewer, commenter, editor
      "notify": true,
      "message": "Please review this proposal"
    }
  ]
}

Response: 201 Created
{
  "shares": [
    {
      "id": "share_123",
      "user": {
        "id": "user_456",
        "email": "colleague@example.com",
        "name": "Jane Smith"
      },
      "role": "editor",
      "created_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

### Create Share Link

```http
POST /v1/documents/{doc_id}/share/link
Content-Type: application/json

Request:
{
  "role": "viewer",  // viewer, commenter
  "expires_at": "2024-02-15T00:00:00Z"
}

Response: 201 Created
{
  "link_id": "link_abc",
  "url": "https://docs.example.com/d/doc_abc123?share=abc123",
  "role": "viewer",
  "expires_at": "2024-02-15T00:00:00Z",
  "created_at": "2024-01-15T10:30:00Z"
}
```

---

## 7. Database Schema

### Core Tables

```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255),
    avatar_url TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_active_at TIMESTAMP WITH TIME ZONE,
    settings JSONB DEFAULT '{}',
    
    INDEX idx_users_email (email)
);

-- Documents table (sharded by owner_id)
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL REFERENCES users(id),
    title VARCHAR(500) NOT NULL DEFAULT 'Untitled Document',
    folder_id UUID,
    current_version BIGINT NOT NULL DEFAULT 1,
    word_count INTEGER DEFAULT 0,
    character_count INTEGER DEFAULT 0,
    is_deleted BOOLEAN DEFAULT FALSE,
    deleted_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    modified_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_edited_by UUID REFERENCES users(id),
    settings JSONB DEFAULT '{}',
    
    INDEX idx_documents_owner (owner_id),
    INDEX idx_documents_folder (folder_id),
    INDEX idx_documents_modified (owner_id, modified_at DESC)
);

-- Document content (current version)
CREATE TABLE document_content (
    document_id UUID PRIMARY KEY REFERENCES documents(id),
    version BIGINT NOT NULL,
    content JSONB NOT NULL,  -- ProseMirror/Quill document model
    content_hash VARCHAR(64),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Document versions (snapshots)
CREATE TABLE document_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(id),
    version BIGINT NOT NULL,
    content JSONB NOT NULL,
    author_id UUID REFERENCES users(id),
    name VARCHAR(255),  -- Named version
    word_count INTEGER,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE (document_id, version),
    INDEX idx_versions_document (document_id, version DESC)
);
```

### Operations Tables (Cassandra/ScyllaDB)

```sql
-- Operations log (time-series, partitioned by document)
CREATE TABLE operations (
    document_id UUID,
    version BIGINT,
    operation_id UUID,
    user_id UUID,
    operation BLOB,  -- Serialized operation
    timestamp TIMESTAMP,
    PRIMARY KEY ((document_id), version, operation_id)
) WITH CLUSTERING ORDER BY (version ASC, operation_id ASC);

-- Pending operations (for offline sync)
CREATE TABLE pending_operations (
    client_id UUID,
    document_id UUID,
    sequence BIGINT,
    operation BLOB,
    created_at TIMESTAMP,
    PRIMARY KEY ((client_id, document_id), sequence)
) WITH CLUSTERING ORDER BY (sequence ASC)
  AND default_time_to_live = 604800;  -- 7 days TTL
```

### Collaboration Tables

```sql
-- Document shares
CREATE TABLE document_shares (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(id),
    user_id UUID REFERENCES users(id),
    email VARCHAR(255),  -- For pending invites
    role VARCHAR(20) NOT NULL,  -- owner, editor, commenter, viewer
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE (document_id, user_id),
    INDEX idx_shares_document (document_id),
    INDEX idx_shares_user (user_id)
);

-- Share links
CREATE TABLE share_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(id),
    short_code VARCHAR(20) NOT NULL UNIQUE,
    role VARCHAR(20) NOT NULL,
    created_by UUID REFERENCES users(id),
    expires_at TIMESTAMP WITH TIME ZONE,
    access_count INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    INDEX idx_share_links_code (short_code),
    INDEX idx_share_links_document (document_id)
);

-- Comments
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(id),
    parent_id UUID REFERENCES comments(id),  -- For replies
    author_id UUID NOT NULL REFERENCES users(id),
    content TEXT NOT NULL,
    anchor_start INTEGER,
    anchor_end INTEGER,
    quoted_text TEXT,
    resolved BOOLEAN DEFAULT FALSE,
    resolved_by UUID REFERENCES users(id),
    resolved_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    INDEX idx_comments_document (document_id),
    INDEX idx_comments_parent (parent_id)
);
```

### Session Tables (Redis)

```
# Active editing sessions
# Key: session:{document_id}:{user_id}
# Value: { client_id, cursor, selection, color, last_active }
# TTL: 5 minutes (refreshed on activity)

# Document state cache
# Key: doc_state:{document_id}
# Value: { version, content_hash, active_users }
# TTL: 1 hour

# Operation buffer (pending sync)
# Key: ops_buffer:{document_id}
# Value: List of operations since last snapshot
# TTL: None (managed by application)

# User presence
# Key: presence:{document_id}
# Value: Set of user_ids currently viewing
# TTL: 30 seconds (refreshed by heartbeat)
```

---

## 8. Document Model (ProseMirror-style)

### Document Structure

```json
{
  "type": "doc",
  "content": [
    {
      "type": "heading",
      "attrs": { "level": 1 },
      "content": [
        { "type": "text", "text": "Project Proposal" }
      ]
    },
    {
      "type": "paragraph",
      "content": [
        { "type": "text", "text": "This is " },
        { 
          "type": "text", 
          "text": "important", 
          "marks": [{ "type": "bold" }] 
        },
        { "type": "text", "text": " text." }
      ]
    },
    {
      "type": "bullet_list",
      "content": [
        {
          "type": "list_item",
          "content": [
            {
              "type": "paragraph",
              "content": [
                { "type": "text", "text": "First item" }
              ]
            }
          ]
        }
      ]
    },
    {
      "type": "image",
      "attrs": {
        "src": "https://storage.example.com/images/abc.png",
        "alt": "Diagram",
        "width": 600,
        "height": 400
      }
    }
  ]
}
```

### Node Types

```typescript
// Node type definitions
interface NodeType {
  name: string;
  content?: string;  // Content expression
  group?: string;
  attrs?: { [key: string]: AttrSpec };
  marks?: string;
  inline?: boolean;
}

const nodeTypes = {
  doc: { content: "block+" },
  paragraph: { content: "inline*", group: "block" },
  heading: { 
    content: "inline*", 
    group: "block",
    attrs: { level: { default: 1 } }
  },
  text: { group: "inline" },
  image: {
    group: "block",
    attrs: {
      src: {},
      alt: { default: null },
      width: { default: null },
      height: { default: null }
    }
  },
  bullet_list: { content: "list_item+", group: "block" },
  ordered_list: { content: "list_item+", group: "block" },
  list_item: { content: "paragraph block*" },
  table: { content: "table_row+", group: "block" },
  table_row: { content: "table_cell+" },
  table_cell: { content: "block+" }
};
```

### Mark Types

```typescript
const markTypes = {
  bold: {},
  italic: {},
  underline: {},
  strike: {},
  code: {},
  link: {
    attrs: {
      href: {},
      title: { default: null }
    }
  },
  text_color: {
    attrs: { color: {} }
  },
  highlight: {
    attrs: { color: {} }
  },
  font_size: {
    attrs: { size: {} }
  },
  font_family: {
    attrs: { family: {} }
  }
};
```

---

## 9. Error Responses

### Standard Error Format

```json
{
  "error": {
    "code": "document_not_found",
    "message": "The requested document does not exist or you don't have access.",
    "details": {
      "document_id": "doc_abc123"
    },
    "request_id": "req_xyz789"
  }
}
```

### Error Codes

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | invalid_operation | Invalid edit operation |
| 400 | version_conflict | Client version behind server |
| 401 | unauthorized | Missing or invalid token |
| 403 | forbidden | No permission for operation |
| 404 | document_not_found | Document doesn't exist |
| 409 | edit_conflict | Concurrent edit conflict |
| 413 | document_too_large | Document exceeds size limit |
| 429 | rate_limited | Too many requests |
| 500 | internal_error | Server error |
| 503 | service_unavailable | Service temporarily unavailable |

### WebSocket Error Messages

```javascript
// Version conflict
{
  type: 'error',
  code: 'version_conflict',
  message: 'Your changes conflict with recent edits',
  server_version: 45,
  client_version: 42,
  recovery: 'sync'  // Client should sync and retry
}

// Permission denied
{
  type: 'error',
  code: 'permission_denied',
  message: 'You no longer have edit access to this document'
}

// Connection limit
{
  type: 'error',
  code: 'connection_limit',
  message: 'Too many editors on this document'
}
```

---

## 10. Idempotency Model

### What is Idempotency?

An operation is **idempotent** if performing it multiple times has the same effect as performing it once. This is critical for collaborative editing to prevent duplicate operations and ensure document consistency.

### Idempotent Operations in Collaborative Document Editor

| Operation | Idempotent? | Mechanism |
|-----------|-------------|-----------|
| **Send Edit Operation** | ✅ Yes | Client sequence number prevents duplicates |
| **Create Document** | ✅ Yes | Idempotency key prevents duplicate documents |
| **Update Permissions** | ✅ Yes | Idempotent state update |
| **Add Comment** | ✅ Yes | Comment ID prevents duplicates |
| **Sync Operations** | ⚠️ At-least-once | Server-side deduplication by (doc_id, client_seq) |

### Idempotency Implementation

**1. Edit Operation with Client Sequence Number:**

```java
@PostMapping("/v1/documents/{docId}/operations")
public ResponseEntity<OperationResponse> applyOperation(
        @PathVariable String docId,
        @RequestBody ApplyOperationRequest request) {
    
    // Check for duplicate operation using client sequence number
    Operation existing = operationRepository.findByDocumentIdAndClientSequence(
        docId, request.getClientId(), request.getClientSequence()
    );
    
    if (existing != null) {
        // Duplicate operation, return existing result
        return ResponseEntity.ok(OperationResponse.from(existing));
    }
    
    // Get current document version
    Document document = documentRepository.findById(docId)
        .orElseThrow(() -> new DocumentNotFoundException(docId));
    
    // Transform operation against server state
    Operation transformed = operationalTransform.transform(
        request.getOperation(),
        document.getVersion(),
        document.getPendingOperations()
    );
    
    // Apply operation
    document = documentService.applyOperation(document, transformed);
    
    // Store operation (idempotent via client sequence)
    Operation operation = Operation.builder()
        .id(generateOperationId())
        .documentId(docId)
        .clientId(request.getClientId())
        .clientSequence(request.getClientSequence())
        .serverSequence(document.getVersion())
        .operation(transformed)
        .appliedAt(Instant.now())
        .build();
    
    operation = operationRepository.save(operation);
    
    // Broadcast to other clients via WebSocket
    websocketService.broadcastOperation(docId, operation);
    
    return ResponseEntity.ok(OperationResponse.from(operation));
}
```

**2. Document Creation with Idempotency Key:**

```java
@PostMapping("/v1/documents")
public ResponseEntity<DocumentResponse> createDocument(
        @RequestBody CreateDocumentRequest request,
        @RequestHeader("X-Idempotency-Key") String idempotencyKey) {
    
    // Check if document already created
    Document existing = documentRepository.findByIdempotencyKey(idempotencyKey);
    if (existing != null) {
        return ResponseEntity.ok(DocumentResponse.from(existing));
    }
    
    // Create new document
    String docId = generateDocumentId();
    Document document = Document.builder()
        .id(docId)
        .idempotencyKey(idempotencyKey)
        .title(request.getTitle())
        .ownerId(request.getUserId())
        .folderId(request.getFolderId())
        .content(request.getInitialContent())
        .version(1)
        .createdAt(Instant.now())
        .modifiedAt(Instant.now())
        .build();
    
    document = documentRepository.save(document);
    
    return ResponseEntity.status(201).body(DocumentResponse.from(document));
}
```

**3. Comment Addition Idempotency:**

```java
@PostMapping("/v1/documents/{docId}/comments")
public ResponseEntity<CommentResponse> addComment(
        @PathVariable String docId,
        @RequestBody AddCommentRequest request,
        @RequestHeader("X-Comment-ID") String commentId) {
    
    // Check if comment already exists
    Comment existing = commentRepository.findByCommentId(commentId);
    if (existing != null) {
        return ResponseEntity.ok(CommentResponse.from(existing));
    }
    
    // Create comment
    Comment comment = Comment.builder()
        .id(commentId)
        .documentId(docId)
        .authorId(request.getUserId())
        .content(request.getContent())
        .position(request.getPosition())
        .createdAt(Instant.now())
        .build();
    
    comment = commentRepository.save(comment);
    
    // Broadcast to other clients
    websocketService.broadcastComment(docId, comment);
    
    return ResponseEntity.status(201).body(CommentResponse.from(comment));
}
```

**4. Permission Update Idempotency:**

```java
@PutMapping("/v1/documents/{docId}/permissions")
public ResponseEntity<PermissionResponse> updatePermissions(
        @PathVariable String docId,
        @RequestBody UpdatePermissionsRequest request,
        @RequestHeader("X-Idempotency-Key") String idempotencyKey) {
    
    // Check if update already processed
    Permission existing = permissionRepository.findByDocumentIdAndIdempotencyKey(
        docId, idempotencyKey
    );
    if (existing != null) {
        return ResponseEntity.ok(PermissionResponse.from(existing));
    }
    
    // Update permissions (idempotent state update)
    Permission permission = permissionRepository.findByDocumentId(docId)
        .orElse(Permission.builder().documentId(docId).build());
    
    permission.setSharedWith(request.getSharedWith());
    permission.setAccessLevel(request.getAccessLevel());
    permission.setIdempotencyKey(idempotencyKey);
    permission.setUpdatedAt(Instant.now());
    
    permission = permissionRepository.save(permission);
    
    return ResponseEntity.ok(PermissionResponse.from(permission));
}
```

### Client Sequence Number Management

**Client-Side:**
```javascript
class OperationQueue {
    constructor() {
        this.clientId = crypto.randomUUID();
        this.clientSequence = 0;
        this.pendingOperations = new Map();
    }
    
    enqueueOperation(operation) {
        const clientSeq = ++this.clientSequence;
        const operationWithSeq = {
            ...operation,
            clientId: this.clientId,
            clientSequence: clientSeq
        };
        
        this.pendingOperations.set(clientSeq, operationWithSeq);
        return operationWithSeq;
    }
    
    acknowledgeOperation(clientSeq) {
        this.pendingOperations.delete(clientSeq);
    }
}
```

### Duplicate Detection Window

| Operation | Deduplication Window | Storage |
|-----------|---------------------|---------|
| Edit Operation | Forever | Cassandra (doc_id, client_id, client_seq unique) |
| Create Document | 24 hours | PostgreSQL (idempotency_key unique index) |
| Add Comment | Forever | PostgreSQL (comment_id unique) |
| Update Permissions | 24 hours | PostgreSQL (idempotency_key unique index) |

### Idempotency Key Storage

```sql
-- Add client sequence tracking to operations
ALTER TABLE operations ADD COLUMN client_id VARCHAR(255);
ALTER TABLE operations ADD COLUMN client_sequence BIGINT;
CREATE UNIQUE INDEX idx_operations_client_seq 
ON operations(document_id, client_id, client_sequence);

-- Add idempotency_key to documents
ALTER TABLE documents ADD COLUMN idempotency_key VARCHAR(255);
CREATE UNIQUE INDEX idx_documents_idempotency_key 
ON documents(idempotency_key) WHERE idempotency_key IS NOT NULL;

-- Comments already have unique comment_id
CREATE UNIQUE INDEX idx_comments_comment_id ON comments(comment_id);
```

