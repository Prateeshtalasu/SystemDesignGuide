# Web Crawler - API & Schema Design

## API Design Philosophy

A web crawler is primarily an internal system, so APIs focus on:

1. **Operational control**: Start/stop crawling, adjust parameters
2. **Monitoring**: Track progress, health, statistics
3. **Data access**: Retrieve crawled content
4. **Configuration**: Manage crawl rules and priorities

---

## Base URL Structure

```
Internal API: https://crawler.internal/api/v1
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
```

**Rate Limits:**

| Endpoint Type | Requests/minute |
|---------------|-----------------|
| Job Management | 100 |
| Data Access | 1000 |
| Monitoring | 500 |

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
| 400 | INVALID_CONFIG | Crawl configuration is invalid |
| 400 | INVALID_URL | URL format is invalid |
| 404 | NOT_FOUND | Resource not found |
| 409 | CONFLICT | Job already exists |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

**Error Response Examples:**

```json
// 400 Bad Request - Invalid config
{
  "error": {
    "code": "INVALID_CONFIG",
    "message": "Crawl configuration is invalid",
    "details": {
      "field": "max_depth",
      "reason": "max_depth must be between 1 and 10"
    },
    "request_id": "req_abc123"
  }
}

// 409 Conflict - Job exists
{
  "error": {
    "code": "CONFLICT",
    "message": "Crawl job with this name already exists",
    "details": {
      "field": "name",
      "reason": "Job 'daily-news-crawl' already exists"
    },
    "request_id": "req_xyz789"
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
| POST /api/v1/jobs | Yes | Idempotency-Key header |
| PUT /api/v1/jobs/{id} | Yes | Idempotency-Key or version-based |
| DELETE /api/v1/jobs/{id} | Yes | Safe to retry (idempotent by design) |
| POST /api/v1/jobs/{id}/pause | Yes | Idempotency-Key header |
| POST /api/v1/jobs/{id}/resume | Yes | Idempotency-Key header |

**Implementation Example:**

```java
@PostMapping("/api/v1/jobs")
public ResponseEntity<JobResponse> createJob(
        @RequestBody CreateJobRequest request,
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {
    
    // Check for existing idempotency key
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        String cachedResponse = redisTemplate.opsForValue().get(cacheKey);
        if (cachedResponse != null) {
            // Return cached response
            JobResponse response = objectMapper.readValue(cachedResponse, JobResponse.class);
            return ResponseEntity.status(response.getStatus()).body(response);
        }
    }
    
    // Check for duplicate job name
    if (jobRepository.existsByName(request.getName())) {
        throw new ConflictException("Job with name already exists");
    }
    
    // Create job
    Job job = jobService.createJob(request, idempotencyKey);
    JobResponse response = JobResponse.from(job);
    
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

## Internal API Endpoints

### 1. Crawl Job Management

**Create Crawl Job**

`POST /api/v1/jobs`

```http
POST /api/v1/jobs HTTP/1.1
Host: crawler.internal
Content-Type: application/json

{
  "name": "daily-news-crawl",
  "seed_urls": [
    "https://cnn.com",
    "https://bbc.com",
    "https://reuters.com"
  ],
  "config": {
    "max_depth": 5,
    "max_pages": 100000,
    "crawl_delay_ms": 1000,
    "respect_robots_txt": true,
    "follow_redirects": true,
    "max_redirects": 5,
    "timeout_seconds": 30,
    "user_agent": "SearchBot/1.0"
  },
  "filters": {
    "allowed_domains": ["cnn.com", "bbc.com", "reuters.com"],
    "url_patterns": [".*\\/news\\/.*", ".*\\/article\\/.*"],
    "excluded_patterns": [".*\\/video\\/.*", ".*\\/gallery\\/.*"]
  },
  "schedule": {
    "type": "recurring",
    "cron": "0 0 * * *"
  }
}
```

**Response (201 Created):**

```json
{
  "job_id": "job_abc123",
  "name": "daily-news-crawl",
  "status": "scheduled",
  "created_at": "2024-01-20T10:00:00Z",
  "next_run": "2024-01-21T00:00:00Z"
}
```

---

**Get Job Status**

`GET /api/v1/jobs/{job_id}`

**Response:**

```json
{
  "job_id": "job_abc123",
  "name": "daily-news-crawl",
  "status": "running",
  "progress": {
    "pages_crawled": 45230,
    "pages_queued": 12450,
    "pages_failed": 234,
    "bytes_downloaded": 4523000000,
    "domains_visited": 3
  },
  "timing": {
    "started_at": "2024-01-20T00:00:00Z",
    "elapsed_seconds": 3600,
    "estimated_completion": "2024-01-20T02:30:00Z"
  },
  "rate": {
    "pages_per_second": 12.5,
    "bytes_per_second": 1250000
  }
}
```

---

**Control Job**

`POST /api/v1/jobs/{job_id}/control`

```json
{
  "action": "pause"  // pause, resume, cancel
}
```

---

### 2. URL Frontier Management

**Add URLs to Frontier**

`POST /api/v1/frontier/urls`

```json
{
  "urls": [
    {
      "url": "https://example.com/page1",
      "priority": 1,
      "metadata": {
        "source": "sitemap",
        "discovered_at": "2024-01-20T10:00:00Z"
      }
    },
    {
      "url": "https://example.com/page2",
      "priority": 5
    }
  ]
}
```

**Response:**

```json
{
  "added": 2,
  "duplicates": 0,
  "blocked": 0
}
```

---

**Get Frontier Statistics**

`GET /api/v1/frontier/stats`

**Response:**

```json
{
  "total_urls": 12345678,
  "by_priority": {
    "1": 100000,
    "2": 500000,
    "3": 2000000,
    "4": 5000000,
    "5": 4745678
  },
  "by_domain": {
    "example.com": 45000,
    "another.com": 32000
  },
  "ready_to_crawl": 234567,
  "delayed": 12000000
}
```

---

### 3. Document Retrieval

**Get Crawled Document**

`GET /api/v1/documents/{url_hash}`

**Response:**

```json
{
  "url": "https://example.com/article/123",
  "url_hash": "abc123def456",
  "crawled_at": "2024-01-20T10:30:00Z",
  "http_status": 200,
  "content_type": "text/html",
  "content_length": 45678,
  "headers": {
    "last-modified": "2024-01-19T15:00:00Z",
    "etag": "\"abc123\"",
    "content-encoding": "gzip"
  },
  "content_hash": "sha256:abcdef...",
  "storage_path": "s3://crawler-data/2024/01/20/abc123.html.gz"
}
```

---

**Search Documents**

`GET /api/v1/documents?domain=example.com&from=2024-01-01&limit=100`

**Response:**

```json
{
  "documents": [
    {
      "url": "https://example.com/page1",
      "url_hash": "abc123",
      "crawled_at": "2024-01-20T10:30:00Z",
      "content_length": 45678
    }
  ],
  "pagination": {
    "total": 5000,
    "offset": 0,
    "limit": 100,
    "next_cursor": "cursor_xyz"
  }
}
```

---

### 4. Robots.txt Management

**Get Robots Rules**

`GET /api/v1/robots/{domain}`

**Response:**

```json
{
  "domain": "example.com",
  "fetched_at": "2024-01-20T00:00:00Z",
  "expires_at": "2024-01-21T00:00:00Z",
  "rules": {
    "user_agents": {
      "*": {
        "allow": ["/public/"],
        "disallow": ["/private/", "/admin/"],
        "crawl_delay": 2
      },
      "SearchBot": {
        "allow": ["/"],
        "disallow": ["/internal/"],
        "crawl_delay": 1
      }
    },
    "sitemaps": [
      "https://example.com/sitemap.xml"
    ]
  }
}
```

---

## Database Schema Design

### Database Choices

| Data Type          | Database       | Rationale                                |
| ------------------ | -------------- | ---------------------------------------- |
| URL Frontier       | Redis + Kafka  | Fast queue operations, persistence       |
| Crawled URLs       | PostgreSQL     | Relational queries, deduplication        |
| Document Metadata  | PostgreSQL     | Structured data, joins                   |
| Raw Content        | S3/HDFS        | Blob storage, cheap at scale             |
| Robots.txt Cache   | Redis          | Fast lookups, TTL support                |
| Bloom Filter       | Redis          | Probabilistic deduplication              |

---

### URL Frontier Schema (Redis + PostgreSQL)

**Redis Structures:**

```
# Priority queue per domain (sorted set)
ZADD frontier:example.com <priority_score> <url_hash>
# Score = priority * 1000000 + timestamp (for ordering)

# Domain crawl timestamps (for politeness)
HSET domain:last_crawl example.com 1705750800

# Ready domains (set of domains ready to crawl)
SADD ready_domains example.com another.com

# Domain metadata
HSET domain:meta:example.com crawl_delay 1000 last_robots_fetch 1705750000
```

**PostgreSQL Schema:**

```sql
CREATE TABLE url_frontier (
    url_hash CHAR(64) PRIMARY KEY,  -- SHA-256 of normalized URL
    url TEXT NOT NULL,
    domain VARCHAR(255) NOT NULL,
    
    -- Priority and scheduling
    priority SMALLINT DEFAULT 5,  -- 1 (highest) to 10 (lowest)
    depth SMALLINT DEFAULT 0,     -- Distance from seed URL
    
    -- Discovery
    discovered_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    discovered_from CHAR(64),     -- Parent URL hash
    
    -- Crawl status
    status VARCHAR(20) DEFAULT 'pending',
    last_crawl_attempt TIMESTAMP WITH TIME ZONE,
    crawl_count SMALLINT DEFAULT 0,
    last_status_code SMALLINT,
    
    -- Scheduling
    next_crawl_after TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CONSTRAINT valid_status CHECK (status IN ('pending', 'crawling', 'completed', 'failed', 'blocked'))
);

-- Index for fetching next URLs to crawl
CREATE INDEX idx_frontier_ready ON url_frontier(domain, priority, next_crawl_after)
    WHERE status = 'pending';

-- Index for domain-based queries
CREATE INDEX idx_frontier_domain ON url_frontier(domain);

-- Index for status monitoring
CREATE INDEX idx_frontier_status ON url_frontier(status);
```

---

### Crawled Documents Schema

```sql
CREATE TABLE crawled_documents (
    url_hash CHAR(64) PRIMARY KEY,
    url TEXT NOT NULL,
    domain VARCHAR(255) NOT NULL,
    
    -- HTTP response
    http_status SMALLINT NOT NULL,
    content_type VARCHAR(100),
    content_length INTEGER,
    content_encoding VARCHAR(50),
    
    -- Headers
    etag VARCHAR(255),
    last_modified TIMESTAMP WITH TIME ZONE,
    
    -- Content
    content_hash CHAR(64),  -- SHA-256 of content
    storage_path TEXT,      -- S3/HDFS path
    
    -- Timestamps
    crawled_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Extracted data
    title TEXT,
    outlinks_count INTEGER DEFAULT 0,
    
    -- Quality signals
    is_duplicate BOOLEAN DEFAULT FALSE,
    duplicate_of CHAR(64)
);

-- Index for content deduplication
CREATE INDEX idx_docs_content_hash ON crawled_documents(content_hash);

-- Index for domain queries
CREATE INDEX idx_docs_domain ON crawled_documents(domain, crawled_at DESC);

-- Index for recent crawls
CREATE INDEX idx_docs_crawled_at ON crawled_documents(crawled_at DESC);
```

---

### Outlinks Table

```sql
CREATE TABLE outlinks (
    id BIGSERIAL PRIMARY KEY,
    source_url_hash CHAR(64) NOT NULL,
    target_url TEXT NOT NULL,
    target_url_hash CHAR(64) NOT NULL,
    
    anchor_text TEXT,
    link_type VARCHAR(20) DEFAULT 'hyperlink',  -- hyperlink, redirect, canonical
    
    discovered_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CONSTRAINT fk_source FOREIGN KEY (source_url_hash) 
        REFERENCES crawled_documents(url_hash)
);

-- Index for discovering new URLs
CREATE INDEX idx_outlinks_target ON outlinks(target_url_hash);

-- Index for link analysis
CREATE INDEX idx_outlinks_source ON outlinks(source_url_hash);
```

---

### Domain Statistics Table

```sql
CREATE TABLE domain_stats (
    domain VARCHAR(255) PRIMARY KEY,
    
    -- Crawl statistics
    pages_crawled BIGINT DEFAULT 0,
    pages_failed BIGINT DEFAULT 0,
    bytes_downloaded BIGINT DEFAULT 0,
    
    -- Timing
    first_crawled_at TIMESTAMP WITH TIME ZONE,
    last_crawled_at TIMESTAMP WITH TIME ZONE,
    avg_response_time_ms INTEGER,
    
    -- Robots.txt
    robots_txt_fetched_at TIMESTAMP WITH TIME ZONE,
    robots_txt_hash CHAR(64),
    crawl_delay_ms INTEGER DEFAULT 1000,
    
    -- Quality
    error_rate FLOAT DEFAULT 0.0,
    is_blocked BOOLEAN DEFAULT FALSE,
    block_reason TEXT
);

CREATE INDEX idx_domain_stats_last_crawl ON domain_stats(last_crawled_at);
```

---

### Crawl Jobs Table

```sql
CREATE TABLE crawl_jobs (
    job_id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    
    -- Configuration (stored as JSON)
    config JSONB NOT NULL,
    seed_urls JSONB NOT NULL,
    filters JSONB,
    
    -- Schedule
    schedule_type VARCHAR(20),  -- one_time, recurring
    cron_expression VARCHAR(100),
    
    -- Status
    status VARCHAR(20) DEFAULT 'created',
    
    -- Progress
    pages_crawled BIGINT DEFAULT 0,
    pages_queued BIGINT DEFAULT 0,
    pages_failed BIGINT DEFAULT 0,
    bytes_downloaded BIGINT DEFAULT 0,
    
    -- Timing
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_job_status CHECK (status IN ('created', 'scheduled', 'running', 'paused', 'completed', 'failed', 'cancelled'))
);
```

---

## Entity Relationship Diagram

```
┌─────────────────────┐
│    crawl_jobs       │
├─────────────────────┤
│ job_id (PK)         │
│ name                │
│ config (JSONB)      │
│ status              │
│ pages_crawled       │
└─────────────────────┘

┌─────────────────────┐       ┌─────────────────────┐
│   url_frontier      │       │  crawled_documents  │
├─────────────────────┤       ├─────────────────────┤
│ url_hash (PK)       │──────>│ url_hash (PK)       │
│ url                 │       │ url                 │
│ domain              │       │ domain              │
│ priority            │       │ http_status         │
│ status              │       │ content_hash        │
│ discovered_from     │       │ storage_path        │
└─────────────────────┘       └──────────┬──────────┘
                                         │
                                         │ 1:N
                                         ▼
                              ┌─────────────────────┐
                              │     outlinks        │
                              ├─────────────────────┤
                              │ id (PK)             │
                              │ source_url_hash(FK) │
                              │ target_url_hash     │
                              │ anchor_text         │
                              └─────────────────────┘

┌─────────────────────┐
│   domain_stats      │
├─────────────────────┤
│ domain (PK)         │
│ pages_crawled       │
│ robots_txt_hash     │
│ crawl_delay_ms      │
│ is_blocked          │
└─────────────────────┘
```

---

## URL Normalization

Consistent URL normalization is critical for deduplication:

```java
public class URLNormalizer {
    
    public String normalize(String url) {
        try {
            URI uri = new URI(url);
            
            // 1. Lowercase scheme and host
            String scheme = uri.getScheme().toLowerCase();
            String host = uri.getHost().toLowerCase();
            
            // 2. Remove default ports
            int port = uri.getPort();
            if ((scheme.equals("http") && port == 80) ||
                (scheme.equals("https") && port == 443)) {
                port = -1;
            }
            
            // 3. Remove trailing slash from path (except root)
            String path = uri.getPath();
            if (path.length() > 1 && path.endsWith("/")) {
                path = path.substring(0, path.length() - 1);
            }
            if (path.isEmpty()) {
                path = "/";
            }
            
            // 4. Sort query parameters
            String query = sortQueryParams(uri.getQuery());
            
            // 5. Remove fragment
            // Fragments are client-side only
            
            // 6. Remove common tracking parameters
            query = removeTrackingParams(query);
            
            // 7. Decode and re-encode path
            path = URLDecoder.decode(path, StandardCharsets.UTF_8);
            path = URLEncoder.encode(path, StandardCharsets.UTF_8)
                   .replace("%2F", "/");
            
            // Reconstruct URL
            StringBuilder normalized = new StringBuilder();
            normalized.append(scheme).append("://").append(host);
            if (port != -1) {
                normalized.append(":").append(port);
            }
            normalized.append(path);
            if (query != null && !query.isEmpty()) {
                normalized.append("?").append(query);
            }
            
            return normalized.toString();
            
        } catch (Exception e) {
            return url;  // Return original if parsing fails
        }
    }
    
    private String removeTrackingParams(String query) {
        if (query == null) return null;
        
        Set<String> trackingParams = Set.of(
            "utm_source", "utm_medium", "utm_campaign", "utm_term", "utm_content",
            "fbclid", "gclid", "ref", "source"
        );
        
        return Arrays.stream(query.split("&"))
            .filter(param -> {
                String key = param.split("=")[0];
                return !trackingParams.contains(key.toLowerCase());
            })
            .collect(Collectors.joining("&"));
    }
}
```

---

## Robots.txt Parsing

```java
public class RobotsParser {
    
    public RobotsRules parse(String robotsTxt, String userAgent) {
        RobotsRules rules = new RobotsRules();
        
        String currentAgent = null;
        boolean matchesOurAgent = false;
        
        for (String line : robotsTxt.split("\n")) {
            line = line.trim();
            
            // Skip comments and empty lines
            if (line.isEmpty() || line.startsWith("#")) continue;
            
            String[] parts = line.split(":", 2);
            if (parts.length != 2) continue;
            
            String directive = parts[0].trim().toLowerCase();
            String value = parts[1].trim();
            
            switch (directive) {
                case "user-agent":
                    currentAgent = value;
                    matchesOurAgent = value.equals("*") || 
                                     userAgent.toLowerCase().contains(value.toLowerCase());
                    break;
                    
                case "disallow":
                    if (matchesOurAgent && !value.isEmpty()) {
                        rules.addDisallow(value);
                    }
                    break;
                    
                case "allow":
                    if (matchesOurAgent) {
                        rules.addAllow(value);
                    }
                    break;
                    
                case "crawl-delay":
                    if (matchesOurAgent) {
                        rules.setCrawlDelay(Integer.parseInt(value));
                    }
                    break;
                    
                case "sitemap":
                    rules.addSitemap(value);
                    break;
            }
        }
        
        return rules;
    }
    
    public boolean isAllowed(RobotsRules rules, String path) {
        // Check allows first (more specific)
        for (String allow : rules.getAllows()) {
            if (pathMatches(path, allow)) {
                return true;
            }
        }
        
        // Then check disallows
        for (String disallow : rules.getDisallows()) {
            if (pathMatches(path, disallow)) {
                return false;
            }
        }
        
        return true;  // Default allow
    }
    
    private boolean pathMatches(String path, String pattern) {
        // Handle wildcards
        if (pattern.contains("*")) {
            String regex = pattern.replace("*", ".*");
            return path.matches(regex);
        }
        
        // Handle $ (end anchor)
        if (pattern.endsWith("$")) {
            return path.equals(pattern.substring(0, pattern.length() - 1));
        }
        
        // Prefix match
        return path.startsWith(pattern);
    }
}
```

---

## Summary

| Component         | Technology/Approach                     |
| ----------------- | --------------------------------------- |
| URL Frontier      | Redis (queue) + PostgreSQL (persistence)|
| Document Metadata | PostgreSQL                              |
| Raw Content       | S3/HDFS                                 |
| Robots.txt Cache  | Redis with TTL                          |
| Deduplication     | Bloom filter + content hash             |
| URL Normalization | Custom normalizer                       |

