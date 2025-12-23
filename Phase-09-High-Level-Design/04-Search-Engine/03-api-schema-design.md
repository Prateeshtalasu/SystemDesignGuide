# Search Engine - API & Schema Design

## API Design Philosophy

Search engine APIs prioritize:

1. **Speed**: Every millisecond matters for user experience
2. **Simplicity**: Easy to use, hard to misuse
3. **Flexibility**: Support various query types and filters
4. **Cacheability**: Design responses for efficient caching

---

## Base URL Structure

```
Production: https://api.search.com/v1
Internal:   https://internal.search.com/v1
```

---

## Core API Endpoints

### 1. Search Query

**Endpoint:** `GET /v1/search`

**Purpose:** Execute a search query and return ranked results

**Request:**

```http
GET /v1/search?q=best+pizza+new+york&page=1&size=10&safe=on HTTP/1.1
Host: api.search.com
Accept: application/json
Accept-Language: en-US
X-User-Location: 40.7128,-74.0060
X-Request-ID: uuid-here
```

**Query Parameters:**

| Parameter   | Type    | Required | Default | Description                        |
| ----------- | ------- | -------- | ------- | ---------------------------------- |
| q           | string  | Yes      | -       | Search query (URL encoded)         |
| page        | int     | No       | 1       | Page number (1-indexed)            |
| size        | int     | No       | 10      | Results per page (max 100)         |
| safe        | string  | No       | on      | Safe search: on, off, moderate     |
| lang        | string  | No       | auto    | Language filter (ISO 639-1)        |
| region      | string  | No       | auto    | Region filter (ISO 3166-1)         |
| freshness   | string  | No       | any     | any, day, week, month, year        |
| site        | string  | No       | -       | Limit to specific domain           |
| filetype    | string  | No       | -       | Filter by file type (pdf, doc)     |

**Response (200 OK):**

```json
{
  "query": {
    "original": "best pizza new york",
    "corrected": null,
    "tokens": ["best", "pizza", "new", "york"],
    "intent": "local_business"
  },
  "results": {
    "total_estimated": 45000000,
    "page": 1,
    "size": 10,
    "items": [
      {
        "position": 1,
        "url": "https://www.timeout.com/newyork/restaurants/best-pizza-in-nyc",
        "title": "Best Pizza in NYC - 25 Top Pizzerias | Time Out",
        "snippet": "Looking for the <em>best pizza in NYC</em>? From classic slices to gourmet pies, these are the top pizzerias in <em>New York</em> City...",
        "displayed_url": "timeout.com › newyork › restaurants",
        "favicon": "https://www.timeout.com/favicon.ico",
        "timestamp": "2024-01-15T10:30:00Z",
        "language": "en",
        "page_rank": 0.85
      },
      {
        "position": 2,
        "url": "https://www.eater.com/maps/best-pizza-nyc",
        "title": "The Best Pizza in New York City - Eater NY",
        "snippet": "The essential guide to <em>New York</em>'s <em>best pizza</em>, from dollar slices to Neapolitan-style pies...",
        "displayed_url": "eater.com › maps",
        "favicon": "https://www.eater.com/favicon.ico",
        "timestamp": "2024-01-18T14:20:00Z",
        "language": "en",
        "page_rank": 0.82
      }
      // ... 8 more results
    ]
  },
  "related_searches": [
    "best pizza nyc 2024",
    "pizza near me",
    "new york style pizza",
    "best pizza manhattan"
  ],
  "spell_suggestions": null,
  "metadata": {
    "search_time_ms": 127,
    "index_time": "2024-01-20T08:00:00Z"
  }
}
```

**Error Responses:**

```json
// 400 Bad Request - Empty query
{
  "error": {
    "code": "EMPTY_QUERY",
    "message": "Search query cannot be empty"
  }
}

// 429 Too Many Requests
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests. Please try again later.",
    "retry_after": 60
  }
}
```

---

### 2. Autocomplete / Query Suggestions

**Endpoint:** `GET /v1/suggest`

**Purpose:** Provide real-time query suggestions as user types

**Request:**

```http
GET /v1/suggest?q=best+piz&limit=8 HTTP/1.1
Host: api.search.com
```

**Response (200 OK):**

```json
{
  "query": "best piz",
  "suggestions": [
    {
      "text": "best pizza near me",
      "score": 0.95,
      "type": "trending"
    },
    {
      "text": "best pizza in new york",
      "score": 0.89,
      "type": "popular"
    },
    {
      "text": "best pizza dough recipe",
      "score": 0.78,
      "type": "popular"
    },
    {
      "text": "best pizza places",
      "score": 0.75,
      "type": "completion"
    }
  ]
}
```

---

### 3. Spell Check

**Endpoint:** `GET /v1/spell`

**Purpose:** Check spelling and suggest corrections

**Request:**

```http
GET /v1/spell?q=reciepe+for+choclate+cake HTTP/1.1
Host: api.search.com
```

**Response (200 OK):**

```json
{
  "original": "reciepe for choclate cake",
  "corrected": "recipe for chocolate cake",
  "corrections": [
    {
      "original": "reciepe",
      "corrected": "recipe",
      "confidence": 0.98
    },
    {
      "original": "choclate",
      "corrected": "chocolate",
      "confidence": 0.99
    }
  ]
}
```

---

### 4. Document Lookup (Internal)

**Endpoint:** `GET /v1/internal/document/{doc_id}`

**Purpose:** Retrieve document metadata by ID (internal use)

**Request:**

```http
GET /v1/internal/document/doc_abc123xyz HTTP/1.1
Host: internal.search.com
Authorization: Bearer internal_api_key
```

**Response (200 OK):**

```json
{
  "doc_id": "doc_abc123xyz",
  "url": "https://example.com/page",
  "title": "Example Page Title",
  "content_hash": "sha256_hash_here",
  "crawled_at": "2024-01-15T10:30:00Z",
  "indexed_at": "2024-01-15T10:35:00Z",
  "page_rank": 0.72,
  "word_count": 1542,
  "language": "en",
  "links_in": 1250,
  "links_out": 45
}
```

---

### 5. Index Status (Internal)

**Endpoint:** `GET /v1/internal/index/status`

**Purpose:** Check index health and statistics

**Response (200 OK):**

```json
{
  "index": {
    "total_documents": 10234567890,
    "total_terms": 12456789,
    "index_size_bytes": 128849018880000,
    "last_updated": "2024-01-20T08:00:00Z"
  },
  "shards": {
    "total": 100,
    "healthy": 98,
    "degraded": 2,
    "offline": 0
  },
  "replication": {
    "factor": 3,
    "in_sync": 294,
    "out_of_sync": 6
  }
}
```

---

## Database Schema Design

### Database Choices

| Data Type          | Database       | Rationale                                |
| ------------------ | -------------- | ---------------------------------------- |
| Inverted Index     | Custom + SSDs  | Optimized for posting list access        |
| Document Metadata  | PostgreSQL     | Relational queries, ACID                 |
| URL Frontier       | Redis + Kafka  | Fast queue operations                    |
| Query Logs         | ClickHouse     | Time-series analytics                    |
| Spell Dictionary   | Redis          | Fast in-memory lookups                   |

---

### Inverted Index Structure

The inverted index is the core data structure. It's stored in a custom binary format optimized for fast access.

**Conceptual Structure:**

```
Term Dictionary (in memory):
┌─────────────┬────────────┬──────────────┐
│ Term        │ Doc Freq   │ Posting Ptr  │
├─────────────┼────────────┼──────────────┤
│ pizza       │ 45,000,000 │ 0x00001000   │
│ restaurant  │ 89,000,000 │ 0x00045000   │
│ new         │ 2.1B       │ 0x00089000   │
│ york        │ 890,000,000│ 0x00123000   │
└─────────────┴────────────┴──────────────┘

Posting List (on disk, memory-mapped):
┌─────────────────────────────────────────────────────────┐
│ Term: "pizza"                                           │
│ Doc Freq: 45,000,000                                    │
├─────────────┬─────────────┬─────────────┬──────────────┤
│ Doc ID      │ Term Freq   │ Positions   │ Score        │
├─────────────┼─────────────┼─────────────┼──────────────┤
│ 12345678    │ 5           │ [12,45,89]  │ 0.85         │
│ 12345679    │ 3           │ [5,23,67]   │ 0.72         │
│ ...         │ ...         │ ...         │ ...          │
└─────────────┴─────────────┴─────────────┴──────────────┘
```

**Compression Techniques:**

1. **Delta Encoding**: Store differences between doc IDs
2. **Variable Byte Encoding**: Use fewer bytes for small numbers
3. **Block Compression**: Compress blocks of posting entries

---

### Document Metadata Table (PostgreSQL)

```sql
CREATE TABLE documents (
    doc_id BIGSERIAL PRIMARY KEY,
    url_hash CHAR(64) UNIQUE NOT NULL,  -- SHA-256 of URL
    url TEXT NOT NULL,
    canonical_url TEXT,
    
    -- Content metadata
    title VARCHAR(500),
    description TEXT,
    content_hash CHAR(64),  -- For duplicate detection
    word_count INTEGER,
    language CHAR(2),
    
    -- Quality signals
    page_rank FLOAT DEFAULT 0.0,
    spam_score FLOAT DEFAULT 0.0,
    links_in INTEGER DEFAULT 0,
    links_out INTEGER DEFAULT 0,
    
    -- Timestamps
    first_crawled_at TIMESTAMP WITH TIME ZONE,
    last_crawled_at TIMESTAMP WITH TIME ZONE,
    last_modified_at TIMESTAMP WITH TIME ZONE,
    indexed_at TIMESTAMP WITH TIME ZONE,
    
    -- Status
    status VARCHAR(20) DEFAULT 'active',  -- active, removed, blocked
    
    CONSTRAINT valid_status CHECK (status IN ('active', 'removed', 'blocked'))
);

-- Indexes
CREATE INDEX idx_documents_url_hash ON documents(url_hash);
CREATE INDEX idx_documents_status ON documents(status) WHERE status = 'active';
CREATE INDEX idx_documents_crawled ON documents(last_crawled_at);
CREATE INDEX idx_documents_pagerank ON documents(page_rank DESC);
```

---

### URL Frontier Table (PostgreSQL + Redis)

```sql
-- PostgreSQL for persistence
CREATE TABLE url_frontier (
    url_id BIGSERIAL PRIMARY KEY,
    url TEXT NOT NULL,
    url_hash CHAR(64) UNIQUE NOT NULL,
    domain VARCHAR(255) NOT NULL,
    
    -- Priority and scheduling
    priority INTEGER DEFAULT 5,  -- 1 (highest) to 10 (lowest)
    scheduled_at TIMESTAMP WITH TIME ZONE,
    
    -- Crawl history
    last_crawled_at TIMESTAMP WITH TIME ZONE,
    crawl_count INTEGER DEFAULT 0,
    last_status_code INTEGER,
    
    -- Source
    discovered_from BIGINT REFERENCES documents(doc_id),
    discovered_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Status
    status VARCHAR(20) DEFAULT 'pending'
);

-- Index for crawler queue
CREATE INDEX idx_frontier_schedule ON url_frontier(scheduled_at, priority)
    WHERE status = 'pending';
CREATE INDEX idx_frontier_domain ON url_frontier(domain);
```

**Redis Queue Structure:**

```
# Priority queues per domain
ZADD frontier:example.com <priority_score> <url_hash>

# Domain crawl timestamps (for politeness)
SET domain:example.com:last_crawl <timestamp>

# Robots.txt cache
SET robots:example.com <robots_txt_content> EX 86400
```

---

### Link Graph Table

```sql
CREATE TABLE link_graph (
    id BIGSERIAL PRIMARY KEY,
    source_doc_id BIGINT NOT NULL,
    target_url_hash CHAR(64) NOT NULL,
    anchor_text TEXT,
    discovered_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- For PageRank calculation
    is_internal BOOLEAN DEFAULT FALSE
);

-- Indexes for PageRank computation
CREATE INDEX idx_links_source ON link_graph(source_doc_id);
CREATE INDEX idx_links_target ON link_graph(target_url_hash);
```

---

### Query Logs Table (ClickHouse)

```sql
CREATE TABLE query_logs (
    query_id UUID,
    query_text String,
    query_tokens Array(String),
    
    -- User context
    user_id String,
    session_id String,
    ip_hash String,
    user_agent String,
    
    -- Location
    country_code FixedString(2),
    region String,
    
    -- Results
    results_count UInt32,
    clicked_positions Array(UInt8),
    
    -- Performance
    search_time_ms UInt32,
    
    -- Timestamp
    timestamp DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (timestamp, query_id);
```

---

### Spell Dictionary (Redis)

```
# Word frequency for spell checking
ZADD spell:words <frequency> <word>

# Edit distance suggestions (precomputed)
HSET spell:edits:resturant restaurant 1
HSET spell:edits:restarant restaurant 1

# Bigram frequencies for context
ZADD spell:bigrams <frequency> "new:york"
ZADD spell:bigrams <frequency> "pizza:restaurant"
```

---

## Entity Relationship Diagram

```
┌─────────────────────┐
│     documents       │
├─────────────────────┤
│ doc_id (PK)         │
│ url_hash (unique)   │
│ url                 │
│ title               │
│ page_rank           │
│ indexed_at          │
└──────────┬──────────┘
           │
           │ 1:N (discovered_from)
           ▼
┌─────────────────────┐
│    url_frontier     │
├─────────────────────┤
│ url_id (PK)         │
│ url_hash (unique)   │
│ domain              │
│ priority            │
│ scheduled_at        │
│ discovered_from(FK) │
└─────────────────────┘

┌─────────────────────┐
│     link_graph      │
├─────────────────────┤
│ id (PK)             │
│ source_doc_id (FK)  │───────┐
│ target_url_hash     │       │
│ anchor_text         │       │
└─────────────────────┘       │
                              │
                              ▼
                    ┌─────────────────────┐
                    │     documents       │
                    │     (source)        │
                    └─────────────────────┘

┌─────────────────────┐       ┌─────────────────────┐
│   inverted_index    │       │    query_logs       │
├─────────────────────┤       ├─────────────────────┤
│ term                │       │ query_id            │
│ doc_frequency       │       │ query_text          │
│ posting_list_ptr    │       │ timestamp           │
│ [doc_id, tf, pos]   │       │ search_time_ms      │
└─────────────────────┘       └─────────────────────┘
```

---

## Ranking Algorithm

### TF-IDF Score

```
TF (Term Frequency) = log(1 + term_count_in_doc)
IDF (Inverse Doc Freq) = log(total_docs / docs_containing_term)
TF-IDF = TF × IDF
```

### BM25 Score (Better than TF-IDF)

```
BM25(D, Q) = Σ IDF(qi) × (tf(qi, D) × (k1 + 1)) / (tf(qi, D) + k1 × (1 - b + b × |D|/avgdl))

Where:
- k1 = 1.2 (term frequency saturation)
- b = 0.75 (document length normalization)
- avgdl = average document length
```

### Combined Ranking Score

```java
public class RankingScorer {
    
    public double calculateScore(Document doc, Query query) {
        // Text relevance (BM25)
        double textScore = calculateBM25(doc, query);
        
        // Link authority (PageRank)
        double authorityScore = doc.getPageRank();
        
        // Freshness (for time-sensitive queries)
        double freshnessScore = calculateFreshness(doc, query);
        
        // Domain trust
        double trustScore = getDomainTrust(doc.getDomain());
        
        // Spam penalty
        double spamPenalty = doc.getSpamScore();
        
        // Weighted combination
        return (textScore * 0.4) + 
               (authorityScore * 0.3) + 
               (freshnessScore * 0.15) + 
               (trustScore * 0.15) - 
               (spamPenalty * 0.5);
    }
    
    private double calculateFreshness(Document doc, Query query) {
        if (!query.isTimeSensitive()) {
            return 0.5; // Neutral
        }
        
        long ageHours = ChronoUnit.HOURS.between(doc.getLastModified(), Instant.now());
        
        // Exponential decay
        return Math.exp(-ageHours / 168.0); // Half-life of 1 week
    }
}
```

---

## Index Sharding Strategy

### Shard by Document ID (Chosen)

```
Shard assignment = hash(doc_id) % num_shards

Pros:
- Even distribution
- Simple to implement
- Easy to add shards

Cons:
- Query must hit all shards
- Network overhead for aggregation
```

### Query Flow with Shards

```
1. Query arrives at coordinator
2. Coordinator broadcasts to all shards
3. Each shard returns top K results
4. Coordinator merges and re-ranks
5. Return final top K to user
```

---

## API Rate Limits

| Tier        | Queries/second | Suggestions/second | Daily Limit |
| ----------- | -------------- | ------------------ | ----------- |
| Free        | 1              | 10                 | 1,000       |
| Developer   | 10             | 100                | 100,000     |
| Business    | 100            | 1,000              | 10,000,000  |
| Enterprise  | Custom         | Custom             | Unlimited   |

---

## Summary

| Component         | Technology/Approach                     |
| ----------------- | --------------------------------------- |
| Search API        | REST with query parameters              |
| Index storage     | Custom binary format + SSD              |
| Document metadata | PostgreSQL                              |
| URL frontier      | PostgreSQL + Redis                      |
| Query logs        | ClickHouse                              |
| Ranking           | BM25 + PageRank + freshness             |
| Sharding          | Hash-based on document ID               |

