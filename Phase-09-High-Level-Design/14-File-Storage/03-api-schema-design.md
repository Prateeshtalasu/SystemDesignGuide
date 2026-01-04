# File Storage System - API & Schema Design

## 1. API Design Principles

### RESTful API Standards

```
Base URL: https://api.cloudstore.com/v1

Authentication: OAuth 2.0 / JWT Bearer Token
Rate Limiting: 1000 requests/minute (standard), 10000/minute (premium)
Versioning: URL path versioning (/v1/, /v2/)
Content-Type: application/json (metadata), multipart/form-data (uploads)
```

### Common Headers

```http
# Request Headers
Authorization: Bearer <access_token>
X-Request-ID: <uuid>           # For tracing
X-Client-Version: 2.5.0        # Client app version
X-Device-ID: <device_uuid>     # For sync

# Response Headers
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1609459200
ETag: "abc123"                 # For caching
X-Request-ID: <uuid>           # Echo back
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
| 400 | FILE_TOO_LARGE | File exceeds size limit |
| 400 | INVALID_FILENAME | Filename is invalid |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | File or folder not found |
| 409 | CONFLICT | File or folder name conflict |
| 413 | PAYLOAD_TOO_LARGE | Request payload too large |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

**Error Response Examples:**

```json
// 400 Bad Request - File too large
{
  "error": {
    "code": "FILE_TOO_LARGE",
    "message": "File exceeds size limit",
    "details": {
      "field": "file",
      "reason": "File size 5GB exceeds limit of 2GB"
    },
    "request_id": "req_abc123"
  }
}

// 409 Conflict - Filename exists
{
  "error": {
    "code": "CONFLICT",
    "message": "File or folder name already exists",
    "details": {
      "field": "filename",
      "reason": "File 'document.pdf' already exists in this folder"
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
| POST /v1/files/upload | Yes | Idempotency-Key header |
| PUT /v1/files/{id} | Yes | Idempotency-Key or version-based |
| DELETE /v1/files/{id} | Yes | Safe to retry (idempotent by design) |
| POST /v1/folders | Yes | Idempotency-Key header |

**Implementation Example:**

```java
@PostMapping("/v1/files/upload")
public ResponseEntity<FileResponse> uploadFile(
        @RequestParam("file") MultipartFile file,
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {
    
    // Check for existing idempotency key
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        String cachedResponse = redisTemplate.opsForValue().get(cacheKey);
        if (cachedResponse != null) {
            FileResponse response = objectMapper.readValue(cachedResponse, FileResponse.class);
            return ResponseEntity.status(response.getStatus()).body(response);
        }
    }
    
    // Upload file
    File uploadedFile = fileService.uploadFile(file, idempotencyKey);
    FileResponse response = FileResponse.from(uploadedFile);
    
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

## 2. File Operations API

### Upload File

```http
POST /v1/files/upload
Content-Type: multipart/form-data

Request:
------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="document.pdf"
Content-Type: application/pdf

<binary file content>
------WebKitFormBoundary
Content-Disposition: form-data; name="parent_id"

folder_abc123
------WebKitFormBoundary
Content-Disposition: form-data; name="conflict_behavior"

rename
------WebKitFormBoundary--

Response: 201 Created
{
  "id": "file_xyz789",
  "name": "document.pdf",
  "path": "/Documents/document.pdf",
  "size": 1048576,
  "mime_type": "application/pdf",
  "hash": "sha256:abc123...",
  "parent_id": "folder_abc123",
  "created_at": "2024-01-15T10:30:00Z",
  "modified_at": "2024-01-15T10:30:00Z",
  "version": 1,
  "owner": {
    "id": "user_123",
    "email": "user@example.com"
  },
  "shared": false,
  "download_url": "https://cdn.cloudstore.com/files/xyz789?token=...",
  "thumbnail_url": "https://cdn.cloudstore.com/thumbs/xyz789.jpg"
}
```

### Chunked Upload (Large Files)

```http
# Step 1: Initialize upload session
POST /v1/files/upload/session
Content-Type: application/json

Request:
{
  "name": "large_video.mp4",
  "size": 5368709120,        // 5 GB
  "parent_id": "folder_abc123",
  "mime_type": "video/mp4",
  "hash": "sha256:abc123..."  // Optional, for deduplication
}

Response: 200 OK
{
  "session_id": "upload_session_123",
  "chunk_size": 10485760,    // 10 MB recommended
  "total_chunks": 512,
  "upload_url": "https://upload.cloudstore.com/sessions/123",
  "expires_at": "2024-01-15T22:30:00Z"
}

# Step 2: Upload chunks
PUT /v1/files/upload/session/{session_id}/chunk/{chunk_number}
Content-Type: application/octet-stream
Content-Range: bytes 0-10485759/5368709120

<binary chunk content>

Response: 200 OK
{
  "chunk_number": 0,
  "received_bytes": 10485760,
  "total_received": 10485760,
  "remaining_chunks": 511
}

# Step 3: Complete upload
POST /v1/files/upload/session/{session_id}/complete
Content-Type: application/json

Request:
{
  "chunk_hashes": [
    "sha256:chunk1hash...",
    "sha256:chunk2hash...",
    // ... all chunk hashes for verification
  ]
}

Response: 201 Created
{
  "id": "file_xyz789",
  "name": "large_video.mp4",
  // ... full file object
}
```

### Download File

```http
GET /v1/files/{file_id}/download
Authorization: Bearer <token>

Response: 302 Redirect
Location: https://cdn.cloudstore.com/files/xyz789?token=...&expires=...

# Or for direct download (small files)
Response: 200 OK
Content-Type: application/pdf
Content-Disposition: attachment; filename="document.pdf"
Content-Length: 1048576

<binary file content>
```

### Get File Metadata

```http
GET /v1/files/{file_id}
Authorization: Bearer <token>

Response: 200 OK
{
  "id": "file_xyz789",
  "name": "document.pdf",
  "path": "/Documents/document.pdf",
  "size": 1048576,
  "mime_type": "application/pdf",
  "hash": "sha256:abc123...",
  "parent_id": "folder_abc123",
  "created_at": "2024-01-15T10:30:00Z",
  "modified_at": "2024-01-15T10:30:00Z",
  "accessed_at": "2024-01-16T08:00:00Z",
  "version": 3,
  "owner": {
    "id": "user_123",
    "email": "user@example.com",
    "name": "John Doe"
  },
  "shared": true,
  "sharing": {
    "link_enabled": true,
    "link_url": "https://cloudstore.com/s/abc123",
    "shared_with": 5
  },
  "permissions": {
    "can_edit": true,
    "can_share": true,
    "can_delete": true
  },
  "preview_url": "https://cdn.cloudstore.com/preview/xyz789",
  "thumbnail_url": "https://cdn.cloudstore.com/thumbs/xyz789.jpg"
}
```

### Update File

```http
PATCH /v1/files/{file_id}
Content-Type: application/json

Request:
{
  "name": "renamed_document.pdf",
  "parent_id": "folder_def456"  // Move to different folder
}

Response: 200 OK
{
  "id": "file_xyz789",
  "name": "renamed_document.pdf",
  "path": "/Projects/renamed_document.pdf",
  // ... updated file object
}
```

### Delete File

```http
DELETE /v1/files/{file_id}
Authorization: Bearer <token>

Query Parameters:
- permanent: boolean (default: false, moves to trash)

Response: 204 No Content

# Or with body for batch delete
DELETE /v1/files
Content-Type: application/json

Request:
{
  "file_ids": ["file_1", "file_2", "file_3"],
  "permanent": false
}

Response: 200 OK
{
  "deleted": ["file_1", "file_2", "file_3"],
  "failed": []
}
```

---

## 3. Folder Operations API

### Create Folder

```http
POST /v1/folders
Content-Type: application/json

Request:
{
  "name": "New Project",
  "parent_id": "folder_root"  // or null for root
}

Response: 201 Created
{
  "id": "folder_abc123",
  "name": "New Project",
  "path": "/New Project",
  "parent_id": "folder_root",
  "created_at": "2024-01-15T10:30:00Z",
  "modified_at": "2024-01-15T10:30:00Z",
  "owner": {
    "id": "user_123",
    "email": "user@example.com"
  },
  "item_count": 0,
  "size": 0
}
```

### List Folder Contents

```http
GET /v1/folders/{folder_id}/items
Authorization: Bearer <token>

Query Parameters:
- limit: integer (default: 100, max: 1000)
- cursor: string (pagination cursor)
- sort: string (name, modified_at, size, type)
- order: string (asc, desc)
- filter: string (files, folders, all)

Response: 200 OK
{
  "items": [
    {
      "id": "folder_xyz",
      "type": "folder",
      "name": "Subfolder",
      "modified_at": "2024-01-15T10:30:00Z",
      "item_count": 5
    },
    {
      "id": "file_abc",
      "type": "file",
      "name": "document.pdf",
      "size": 1048576,
      "mime_type": "application/pdf",
      "modified_at": "2024-01-15T10:30:00Z",
      "thumbnail_url": "https://cdn.cloudstore.com/thumbs/abc.jpg"
    }
  ],
  "cursor": "eyJsYXN0X2lkIjoiZmlsZV9hYmMifQ==",
  "has_more": true,
  "total_count": 150
}
```

---

## 4. Sync API

### Get Changes (Delta Sync)

```http
GET /v1/sync/delta
Authorization: Bearer <token>

Query Parameters:
- cursor: string (last sync cursor, empty for initial sync)
- path_prefix: string (optional, sync specific folder)
- limit: integer (default: 500, max: 2000)

Response: 200 OK
{
  "entries": [
    {
      "type": "file",
      "action": "created",
      "id": "file_xyz789",
      "path": "/Documents/new_file.pdf",
      "server_modified": "2024-01-15T10:30:00Z",
      "size": 1048576,
      "hash": "sha256:abc123...",
      "rev": "015f8e9c0000000000001"
    },
    {
      "type": "file",
      "action": "modified",
      "id": "file_abc123",
      "path": "/Documents/updated.docx",
      "server_modified": "2024-01-15T10:35:00Z",
      "size": 2097152,
      "hash": "sha256:def456...",
      "rev": "015f8e9c0000000000002"
    },
    {
      "type": "file",
      "action": "deleted",
      "id": "file_old123",
      "path": "/Documents/deleted.txt"
    },
    {
      "type": "folder",
      "action": "created",
      "id": "folder_new",
      "path": "/Documents/New Folder"
    }
  ],
  "cursor": "AAGpZmVkY2JhOTg3NjU0MzIx",
  "has_more": false,
  "reset": false  // true if client should do full sync
}
```

### Long Poll for Changes

```http
POST /v1/sync/longpoll
Content-Type: application/json

Request:
{
  "cursor": "AAGpZmVkY2JhOTg3NjU0MzIx",
  "timeout": 30  // seconds
}

Response: 200 OK
{
  "changes": true  // or false if timeout
}

# If changes=true, client calls GET /v1/sync/delta
```

### Upload Sync State

```http
POST /v1/sync/commit
Content-Type: application/json

Request:
{
  "device_id": "device_abc123",
  "entries": [
    {
      "path": "/Documents/local_file.pdf",
      "action": "upload",
      "local_modified": "2024-01-15T10:30:00Z",
      "size": 1048576,
      "hash": "sha256:abc123..."
    },
    {
      "path": "/Documents/deleted_locally.txt",
      "action": "delete"
    }
  ]
}

Response: 200 OK
{
  "results": [
    {
      "path": "/Documents/local_file.pdf",
      "status": "upload_required",
      "upload_url": "https://upload.cloudstore.com/sync/..."
    },
    {
      "path": "/Documents/deleted_locally.txt",
      "status": "conflict",
      "conflict_type": "modified_on_server",
      "server_version": {
        "modified_at": "2024-01-15T10:35:00Z",
        "hash": "sha256:different..."
      }
    }
  ]
}
```

---

## 5. Sharing API

### Create Share Link

```http
POST /v1/files/{file_id}/share/link
Content-Type: application/json

Request:
{
  "access_level": "viewer",     // viewer, editor
  "password": "optional_password",
  "expires_at": "2024-02-15T00:00:00Z",
  "download_limit": 100,        // optional
  "allow_download": true
}

Response: 201 Created
{
  "link_id": "link_abc123",
  "url": "https://cloudstore.com/s/abc123",
  "short_url": "https://cs.co/abc123",
  "access_level": "viewer",
  "password_protected": true,
  "expires_at": "2024-02-15T00:00:00Z",
  "download_limit": 100,
  "download_count": 0,
  "created_at": "2024-01-15T10:30:00Z"
}
```

### Share with Users

```http
POST /v1/files/{file_id}/share/users
Content-Type: application/json

Request:
{
  "shares": [
    {
      "email": "colleague@company.com",
      "access_level": "editor",
      "notify": true,
      "message": "Please review this document"
    },
    {
      "email": "client@external.com",
      "access_level": "viewer",
      "notify": true
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
        "email": "colleague@company.com",
        "name": "Jane Doe"
      },
      "access_level": "editor",
      "status": "active"
    },
    {
      "id": "share_124",
      "email": "client@external.com",
      "access_level": "viewer",
      "status": "pending"  // User not registered
    }
  ]
}
```

### List Shares

```http
GET /v1/files/{file_id}/shares
Authorization: Bearer <token>

Response: 200 OK
{
  "owner": {
    "id": "user_123",
    "email": "owner@company.com",
    "name": "John Doe"
  },
  "link_shares": [
    {
      "link_id": "link_abc123",
      "url": "https://cloudstore.com/s/abc123",
      "access_level": "viewer",
      "created_at": "2024-01-15T10:30:00Z",
      "view_count": 25
    }
  ],
  "user_shares": [
    {
      "id": "share_123",
      "user": {
        "id": "user_456",
        "email": "colleague@company.com"
      },
      "access_level": "editor",
      "shared_at": "2024-01-15T10:30:00Z",
      "last_accessed": "2024-01-16T08:00:00Z"
    }
  ]
}
```

---

## 6. Search API

### Search Files

```http
GET /v1/search
Authorization: Bearer <token>

Query Parameters:
- q: string (search query)
- path: string (limit to path prefix)
- type: string (file, folder, all)
- file_types: string (pdf,docx,jpg - comma separated)
- modified_after: ISO8601 timestamp
- modified_before: ISO8601 timestamp
- size_min: integer (bytes)
- size_max: integer (bytes)
- owner: string (user_id or "me")
- shared: boolean
- limit: integer (default: 50)
- cursor: string

Response: 200 OK
{
  "results": [
    {
      "id": "file_xyz789",
      "type": "file",
      "name": "quarterly_report.pdf",
      "path": "/Documents/Reports/quarterly_report.pdf",
      "size": 1048576,
      "mime_type": "application/pdf",
      "modified_at": "2024-01-15T10:30:00Z",
      "highlight": {
        "content": "...revenue growth in <em>Q4</em> exceeded expectations..."
      },
      "score": 0.95
    }
  ],
  "cursor": "eyJvZmZzZXQiOjUwfQ==",
  "has_more": true,
  "total_count": 150,
  "query_time_ms": 45
}
```

---

## 7. Version History API

### List Versions

```http
GET /v1/files/{file_id}/versions
Authorization: Bearer <token>

Query Parameters:
- limit: integer (default: 20)
- cursor: string

Response: 200 OK
{
  "versions": [
    {
      "version_id": "v_current",
      "version_number": 5,
      "size": 1048576,
      "modified_at": "2024-01-15T10:30:00Z",
      "modified_by": {
        "id": "user_123",
        "name": "John Doe"
      },
      "is_current": true
    },
    {
      "version_id": "v_abc123",
      "version_number": 4,
      "size": 1000000,
      "modified_at": "2024-01-14T15:00:00Z",
      "modified_by": {
        "id": "user_456",
        "name": "Jane Doe"
      },
      "is_current": false
    }
  ],
  "cursor": null,
  "has_more": false
}
```

### Restore Version

```http
POST /v1/files/{file_id}/versions/{version_id}/restore
Authorization: Bearer <token>

Response: 200 OK
{
  "id": "file_xyz789",
  "version": 6,  // New version created from restored
  // ... full file object
}
```

---

## 8. Database Schema

### Core Tables

```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    email_verified BOOLEAN DEFAULT FALSE,
    password_hash VARCHAR(255),
    name VARCHAR(255),
    profile_picture_url TEXT,
    storage_quota_bytes BIGINT DEFAULT 16106127360,  -- 15 GB
    storage_used_bytes BIGINT DEFAULT 0,
    account_type VARCHAR(50) DEFAULT 'free',  -- free, pro, business, enterprise
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_login_at TIMESTAMP WITH TIME ZONE,
    settings JSONB DEFAULT '{}',
    
    INDEX idx_users_email (email)
);

-- Files table (sharded by owner_id)
CREATE TABLE files (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL REFERENCES users(id),
    parent_id UUID REFERENCES folders(id),
    name VARCHAR(255) NOT NULL,
    path TEXT NOT NULL,  -- Full path for search/display
    path_lower TEXT NOT NULL,  -- Lowercase for case-insensitive operations
    size_bytes BIGINT NOT NULL,
    mime_type VARCHAR(255),
    content_hash VARCHAR(64) NOT NULL,  -- SHA-256 of content
    block_hashes TEXT[],  -- Hashes of each block for dedup
    storage_key TEXT NOT NULL,  -- S3 object key
    version INTEGER DEFAULT 1,
    is_deleted BOOLEAN DEFAULT FALSE,
    deleted_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    modified_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    accessed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Sync metadata
    rev VARCHAR(50) NOT NULL,  -- Revision ID for sync
    client_modified_at TIMESTAMP WITH TIME ZONE,
    
    UNIQUE (owner_id, path_lower),
    INDEX idx_files_owner (owner_id),
    INDEX idx_files_parent (parent_id),
    INDEX idx_files_path (owner_id, path_lower),
    INDEX idx_files_hash (content_hash),
    INDEX idx_files_modified (owner_id, modified_at DESC),
    INDEX idx_files_deleted (owner_id, is_deleted, deleted_at)
);

-- Folders table
CREATE TABLE folders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL REFERENCES users(id),
    parent_id UUID REFERENCES folders(id),
    name VARCHAR(255) NOT NULL,
    path TEXT NOT NULL,
    path_lower TEXT NOT NULL,
    is_deleted BOOLEAN DEFAULT FALSE,
    deleted_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    modified_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Sync metadata
    rev VARCHAR(50) NOT NULL,
    
    UNIQUE (owner_id, path_lower),
    INDEX idx_folders_owner (owner_id),
    INDEX idx_folders_parent (parent_id),
    INDEX idx_folders_path (owner_id, path_lower)
);

-- File versions table
CREATE TABLE file_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id UUID NOT NULL REFERENCES files(id),
    version_number INTEGER NOT NULL,
    size_bytes BIGINT NOT NULL,
    content_hash VARCHAR(64) NOT NULL,
    storage_key TEXT NOT NULL,
    modified_by UUID REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE (file_id, version_number),
    INDEX idx_versions_file (file_id, version_number DESC)
);

-- File blocks table (for chunked storage and deduplication)
CREATE TABLE file_blocks (
    hash VARCHAR(64) PRIMARY KEY,  -- SHA-256 of block
    size_bytes INTEGER NOT NULL,
    storage_key TEXT NOT NULL,
    reference_count INTEGER DEFAULT 1,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- File-to-block mapping
CREATE TABLE file_block_map (
    file_id UUID NOT NULL REFERENCES files(id),
    block_hash VARCHAR(64) NOT NULL REFERENCES file_blocks(hash),
    block_index INTEGER NOT NULL,
    
    PRIMARY KEY (file_id, block_index),
    INDEX idx_block_map_hash (block_hash)
);
```

### Sharing Tables

```sql
-- Share links table
CREATE TABLE share_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id UUID REFERENCES files(id),
    folder_id UUID REFERENCES folders(id),
    created_by UUID NOT NULL REFERENCES users(id),
    short_code VARCHAR(20) NOT NULL UNIQUE,
    access_level VARCHAR(20) NOT NULL,  -- viewer, editor
    password_hash VARCHAR(255),
    expires_at TIMESTAMP WITH TIME ZONE,
    download_limit INTEGER,
    download_count INTEGER DEFAULT 0,
    view_count INTEGER DEFAULT 0,
    allow_download BOOLEAN DEFAULT TRUE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CHECK (file_id IS NOT NULL OR folder_id IS NOT NULL),
    INDEX idx_share_links_code (short_code),
    INDEX idx_share_links_file (file_id),
    INDEX idx_share_links_folder (folder_id)
);

-- User shares table
CREATE TABLE user_shares (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id UUID REFERENCES files(id),
    folder_id UUID REFERENCES folders(id),
    shared_by UUID NOT NULL REFERENCES users(id),
    shared_with UUID REFERENCES users(id),
    shared_with_email VARCHAR(255),  -- For pending invites
    access_level VARCHAR(20) NOT NULL,
    status VARCHAR(20) DEFAULT 'active',  -- active, pending, revoked
    notify_on_change BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    accepted_at TIMESTAMP WITH TIME ZONE,
    last_accessed_at TIMESTAMP WITH TIME ZONE,
    
    CHECK (file_id IS NOT NULL OR folder_id IS NOT NULL),
    CHECK (shared_with IS NOT NULL OR shared_with_email IS NOT NULL),
    UNIQUE (file_id, shared_with),
    UNIQUE (folder_id, shared_with),
    INDEX idx_user_shares_file (file_id),
    INDEX idx_user_shares_folder (folder_id),
    INDEX idx_user_shares_user (shared_with),
    INDEX idx_user_shares_email (shared_with_email)
);

-- Team/Organization tables
CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id),
    storage_quota_bytes BIGINT,
    storage_used_bytes BIGINT DEFAULT 0,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    INDEX idx_teams_owner (owner_id)
);

CREATE TABLE team_members (
    team_id UUID NOT NULL REFERENCES teams(id),
    user_id UUID NOT NULL REFERENCES users(id),
    role VARCHAR(50) NOT NULL,  -- owner, admin, member
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    PRIMARY KEY (team_id, user_id),
    INDEX idx_team_members_user (user_id)
);

CREATE TABLE team_folders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    folder_id UUID NOT NULL REFERENCES folders(id),
    default_access_level VARCHAR(20) DEFAULT 'viewer',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE (team_id, folder_id),
    INDEX idx_team_folders_team (team_id)
);
```

### Activity and Sync Tables

```sql
-- Activity log table (time-series)
CREATE TABLE activity_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    file_id UUID,
    folder_id UUID,
    action VARCHAR(50) NOT NULL,  -- upload, download, share, delete, etc.
    details JSONB,
    ip_address INET,
    user_agent TEXT,
    device_id VARCHAR(100),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Partition by time for efficient querying and retention
CREATE INDEX idx_activity_user_time ON activity_log (user_id, created_at DESC);
CREATE INDEX idx_activity_file ON activity_log (file_id, created_at DESC);

-- Sync cursors table
CREATE TABLE sync_cursors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    device_id VARCHAR(100) NOT NULL,
    cursor_value TEXT NOT NULL,
    path_prefix TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE (user_id, device_id, path_prefix),
    INDEX idx_sync_cursors_user (user_id)
);

-- Device registry
CREATE TABLE devices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    device_id VARCHAR(100) NOT NULL,
    device_name VARCHAR(255),
    device_type VARCHAR(50),  -- desktop, mobile, web
    platform VARCHAR(50),  -- windows, macos, linux, ios, android
    client_version VARCHAR(50),
    last_sync_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE (user_id, device_id),
    INDEX idx_devices_user (user_id)
);
```

### Upload Session Tables

```sql
-- Upload sessions for chunked uploads
CREATE TABLE upload_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    file_name VARCHAR(255) NOT NULL,
    file_size BIGINT NOT NULL,
    mime_type VARCHAR(255),
    parent_id UUID,
    chunk_size INTEGER NOT NULL,
    total_chunks INTEGER NOT NULL,
    received_chunks INTEGER DEFAULT 0,
    status VARCHAR(20) DEFAULT 'active',  -- active, completed, expired, failed
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    completed_at TIMESTAMP WITH TIME ZONE,
    
    INDEX idx_upload_sessions_user (user_id),
    INDEX idx_upload_sessions_status (status, expires_at)
);

-- Chunk tracking
CREATE TABLE upload_chunks (
    session_id UUID NOT NULL REFERENCES upload_sessions(id),
    chunk_number INTEGER NOT NULL,
    storage_key TEXT NOT NULL,
    size_bytes INTEGER NOT NULL,
    hash VARCHAR(64) NOT NULL,
    uploaded_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    PRIMARY KEY (session_id, chunk_number)
);
```

---

## 9. Error Responses

### Standard Error Format

```json
{
  "error": {
    "code": "file_not_found",
    "message": "The requested file does not exist or you don't have access to it.",
    "details": {
      "file_id": "file_xyz789"
    },
    "request_id": "req_abc123"
  }
}
```

### Error Codes

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | invalid_request | Malformed request |
| 400 | invalid_path | Invalid file/folder path |
| 400 | name_conflict | File/folder with name already exists |
| 401 | unauthorized | Missing or invalid token |
| 403 | forbidden | No permission for operation |
| 403 | quota_exceeded | Storage quota exceeded |
| 404 | file_not_found | File doesn't exist |
| 404 | folder_not_found | Folder doesn't exist |
| 409 | conflict | Sync conflict detected |
| 413 | file_too_large | File exceeds size limit |
| 429 | rate_limited | Too many requests |
| 500 | internal_error | Server error |
| 503 | service_unavailable | Service temporarily unavailable |

---

## 10. Idempotency Model

### What is Idempotency?

An operation is **idempotent** if performing it multiple times has the same effect as performing it once. This is critical for file uploads, chunked uploads, and file operations to prevent duplicates and handle network retries.

### Idempotent Operations in File Storage

| Operation | Idempotent? | Mechanism |
|-----------|-------------|-----------|
| **Initiate Upload Session** | ✅ Yes | Idempotency key prevents duplicate sessions |
| **Upload Chunk** | ✅ Yes | Chunk hash + position ensures idempotency |
| **Complete Upload** | ✅ Yes | Session ID + file hash prevents duplicates |
| **Delete File** | ✅ Yes | DELETE is idempotent (safe to retry) |
| **Update File Metadata** | ✅ Yes | Version-based updates prevent conflicts |
| **Sync Operations** | ⚠️ At-least-once | Client-side deduplication by event ID |

### Idempotency Implementation

**1. Upload Session Initiation with Idempotency Key:**

```java
@PostMapping("/v1/files/upload/session")
public ResponseEntity<UploadSessionResponse> initiateUploadSession(
        @RequestBody InitiateUploadRequest request,
        @RequestHeader("X-Idempotency-Key") String idempotencyKey) {
    
    // Check if session already exists
    UploadSession existing = uploadSessionRepository.findByIdempotencyKey(idempotencyKey);
    if (existing != null) {
        return ResponseEntity.ok(UploadSessionResponse.from(existing));
    }
    
    // Check for content deduplication (if hash provided)
    if (request.getHash() != null) {
        File existingFile = fileRepository.findByContentHash(request.getHash());
        if (existingFile != null && existingFile.getOwnerId().equals(request.getUserId())) {
            // Same content already uploaded by same user, return existing file
            return ResponseEntity.ok(UploadSessionResponse.fromExisting(existingFile));
        }
    }
    
    // Create new upload session
    String sessionId = generateSessionId();
    UploadSession session = UploadSession.builder()
        .sessionId(sessionId)
        .idempotencyKey(idempotencyKey)
        .userId(request.getUserId())
        .fileName(request.getName())
        .fileSize(request.getSize())
        .contentHash(request.getHash())
        .chunkSize(calculateChunkSize(request.getSize()))
        .status(UploadStatus.INITIATED)
        .expiresAt(Instant.now().plus(24, ChronoUnit.HOURS))
        .build();
    
    session = uploadSessionRepository.save(session);
    
    return ResponseEntity.ok(UploadSessionResponse.from(session));
}
```

**2. Chunk Upload Idempotency:**

```java
@PutMapping("/v1/files/upload/{sessionId}/chunk")
public ResponseEntity<ChunkResponse> uploadChunk(
        @PathVariable String sessionId,
        @RequestParam int chunkIndex,
        @RequestHeader("Content-MD5") String chunkHash,
        @RequestBody byte[] chunkData) {
    
    // Verify chunk hash
    String calculatedHash = DigestUtils.md5Hex(chunkData);
    if (!calculatedHash.equals(chunkHash)) {
        throw new InvalidChunkException("Chunk hash mismatch");
    }
    
    // Check if chunk already uploaded
    ChunkStatus existing = chunkRepository.findBySessionIdAndIndex(sessionId, chunkIndex);
    if (existing != null && existing.getHash().equals(chunkHash)) {
        // Chunk already uploaded with same hash, return existing status
        return ResponseEntity.ok(ChunkResponse.from(existing));
    }
    
    // Upload chunk to S3 (idempotent: same key = overwrite)
    String chunkKey = String.format("uploads/%s/chunk_%d", sessionId, chunkIndex);
    s3Service.putObject(chunkKey, chunkData);
    
    // Record chunk completion
    ChunkStatus chunkStatus = ChunkStatus.builder()
        .sessionId(sessionId)
        .chunkIndex(chunkIndex)
        .hash(chunkHash)
        .size(chunkData.length)
        .status(ChunkStatus.COMPLETED)
        .uploadedAt(Instant.now())
        .build();
    
    chunkRepository.save(chunkStatus);
    
    return ResponseEntity.ok(ChunkResponse.from(chunkStatus));
}
```

**3. Complete Upload Idempotency:**

```java
@PostMapping("/v1/files/upload/{sessionId}/complete")
public ResponseEntity<FileResponse> completeUpload(
        @PathVariable String sessionId,
        @RequestHeader("X-Idempotency-Key") String idempotencyKey) {
    
    // Check if file already created from this session
    File existing = fileRepository.findByUploadSessionId(sessionId);
    if (existing != null) {
        return ResponseEntity.ok(FileResponse.from(existing));
    }
    
    // Verify all chunks uploaded
    UploadSession session = uploadSessionRepository.findById(sessionId)
        .orElseThrow(() -> new SessionNotFoundException(sessionId));
    
    List<ChunkStatus> chunks = chunkRepository.findBySessionId(sessionId);
    if (chunks.size() != calculateExpectedChunks(session.getFileSize(), session.getChunkSize())) {
        throw new IncompleteUploadException("Not all chunks uploaded");
    }
    
    // Assemble file from chunks (idempotent: S3 multipart upload)
    String fileKey = assembleFileFromChunks(sessionId, chunks);
    
    // Calculate final file hash
    String finalHash = calculateFileHash(fileKey);
    
    // Check for content deduplication
    File duplicate = fileRepository.findByContentHash(finalHash);
    if (duplicate != null) {
        // Same content exists, create reference instead
        File file = createFileReference(session, duplicate, fileKey);
        return ResponseEntity.ok(FileResponse.from(file));
    }
    
    // Create new file record
    File file = File.builder()
        .id(generateFileId())
        .uploadSessionId(sessionId)
        .idempotencyKey(idempotencyKey)
        .ownerId(session.getUserId())
        .name(session.getFileName())
        .size(session.getFileSize())
        .contentHash(finalHash)
        .storageKey(fileKey)
        .status(FileStatus.ACTIVE)
        .createdAt(Instant.now())
        .build();
    
    file = fileRepository.save(file);
    
    return ResponseEntity.status(201).body(FileResponse.from(file));
}
```

**4. Sync Operation Deduplication:**

```java
// Deduplicate sync events by (user_id, device_id, event_id)
public void processSyncEvent(SyncEvent event) {
    String dedupKey = String.format("sync:%s:%s:%s",
        event.getUserId(),
        event.getDeviceId(),
        event.getEventId()
    );
    
    // Set if absent (idempotent)
    Boolean isNew = redisTemplate.opsForValue()
        .setIfAbsent(dedupKey, "1", Duration.ofHours(24));
    
    if (Boolean.TRUE.equals(isNew)) {
        // New event, process it
        syncService.applyEvent(event);
    }
    // Duplicate event, ignore
}
```

### Idempotency Key Generation

**Client-Side (Recommended):**
```javascript
// Generate UUID v4 for each upload session
const idempotencyKey = crypto.randomUUID();

fetch('/v1/files/upload/session', {
    method: 'POST',
    headers: {
        'Authorization': `Bearer ${token}`,
        'X-Idempotency-Key': idempotencyKey
    },
    body: sessionData
});
```

**Server-Side (Fallback):**
```java
// If client doesn't provide key, generate from request hash
String idempotencyKey = DigestUtils.sha256Hex(
    request.getUserId() + 
    request.getName() +
    request.getSize() +
    request.getHash()
);
```

### Duplicate Detection Window

| Operation | Deduplication Window | Storage |
|-----------|---------------------|---------|
| Upload Session | 24 hours | PostgreSQL (idempotency_key unique index) |
| Chunk Upload | Forever | S3 (same key = overwrite) |
| Complete Upload | 24 hours | PostgreSQL (upload_session_id unique) |
| Content Deduplication | Forever | PostgreSQL (content_hash index) |
| Sync Events | 24 hours | Redis |

### Idempotency Key Storage

```sql
-- Add idempotency_key to upload_sessions
ALTER TABLE upload_sessions ADD COLUMN idempotency_key VARCHAR(255);
CREATE UNIQUE INDEX idx_upload_sessions_idempotency_key 
ON upload_sessions(idempotency_key) WHERE idempotency_key IS NOT NULL;

-- Add content_hash for deduplication
ALTER TABLE files ADD COLUMN content_hash VARCHAR(64);
CREATE INDEX idx_files_content_hash ON files(content_hash);
```

---

## 11. Webhook Events

### Event Types

```json
// File created
{
  "event_type": "file.created",
  "event_id": "evt_abc123",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "file": {
      "id": "file_xyz789",
      "name": "document.pdf",
      "path": "/Documents/document.pdf"
    },
    "user": {
      "id": "user_123",
      "email": "user@example.com"
    }
  }
}

// File shared
{
  "event_type": "file.shared",
  "event_id": "evt_def456",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "file": {
      "id": "file_xyz789",
      "name": "document.pdf"
    },
    "shared_by": {
      "id": "user_123",
      "email": "owner@example.com"
    },
    "shared_with": {
      "id": "user_456",
      "email": "colleague@example.com"
    },
    "access_level": "editor"
  }
}
```

### Webhook Configuration

```http
POST /v1/webhooks
Content-Type: application/json

Request:
{
  "url": "https://myapp.com/webhooks/cloudstore",
  "events": ["file.created", "file.modified", "file.deleted", "file.shared"],
  "secret": "webhook_secret_for_signature"
}

Response: 201 Created
{
  "id": "webhook_abc123",
  "url": "https://myapp.com/webhooks/cloudstore",
  "events": ["file.created", "file.modified", "file.deleted", "file.shared"],
  "status": "active",
  "created_at": "2024-01-15T10:30:00Z"
}
```

