# Search Engine - Data Model & Architecture

## Component Overview

Before looking at diagrams, let's understand each component and why it exists.

### Components Explained

| Component | Purpose | Why It Exists |
|-----------|---------|---------------|
| **Web Crawler** | Discovers and downloads web pages | Can't index what we haven't fetched |
| **URL Frontier** | Queue of URLs to crawl | Manages crawl priority and scheduling |
| **Document Processor** | Parses HTML, extracts content | Raw HTML isn't searchable |
| **Indexer** | Builds inverted index | Enables fast keyword lookup |
| **Index Shards** | Distributed index storage | Single machine can't hold 120 TB |
| **Query Processor** | Parses and expands queries | Transforms user input into searchable terms |
| **Ranking Engine** | Scores and orders results | Relevance is key to user satisfaction |
| **Results Cache** | Caches popular query results | Reduces latency for common queries |
| **PageRank Computer** | Calculates link-based authority | Quality signal for ranking |

---

## Database Choices

| Data Type | Database | Rationale |
|-----------|----------|-----------|
| Inverted Index | Custom + SSDs | Optimized for posting list access |
| Document Metadata | PostgreSQL | Relational queries, ACID |
| URL Frontier | Redis + Kafka | Fast queue operations |
| Query Logs | ClickHouse | Time-series analytics |
| Spell Dictionary | Redis | Fast in-memory lookups |

### Why This Combination?

**Inverted Index (Custom Binary):**
- Most critical component, needs maximum performance
- Standard databases can't handle posting list access patterns
- Memory-mapped files for fast random access
- Custom compression (delta + VByte) for 4-8x size reduction

**PostgreSQL (Document Metadata):**
- Need ACID for URL deduplication
- Complex queries for crawl scheduling
- Relational model fits document-link relationships

**Redis (Frontier + Spell):**
- Sub-millisecond lookups for politeness delays
- Sorted sets for priority queues
- In-memory speed for spell checking

**ClickHouse (Query Logs):**
- Columnar storage for analytics
- Handles billions of query logs
- Fast aggregations for trending queries

---

## Consistency Model

**CAP Theorem Tradeoff:**

We choose **Availability + Partition Tolerance (AP)**:
- **Availability**: Search must always respond (even with stale index)
- **Partition Tolerance**: System continues operating during network partitions
- **Consistency**: Sacrificed (index may be hours/days stale)

**Why AP over CP?**
- Search results don't need to be real-time (hours/days delay acceptable)
- Better to serve stale results than fail search requests
- System must always respond (high availability requirement)
- During partitions, we prefer serving stale index over failing

**ACID vs BASE:**

**ACID (Strong Consistency) for:**
- URL deduplication (PostgreSQL, must prevent duplicate crawls)
- Document metadata updates (PostgreSQL transactions)
- Crawl scheduling (PostgreSQL, prevent duplicate scheduling)

**BASE (Eventual Consistency) for:**
- Index updates (indexed documents may be hours/days old)
- Search results (may not include latest web pages)
- PageRank calculations (updated periodically, not real-time)
- Query logs (processed in batches)

**Per-Operation Consistency Guarantees:**

| Operation | Consistency Level | Guarantee |
|-----------|------------------|-----------|
| Search query | Eventual | Results may be hours/days stale |
| Index update | Eventual | New pages indexed within hours/days |
| URL deduplication | Strong | Prevents duplicate crawls (ACID) |
| Crawl scheduling | Strong | Prevents duplicate scheduling (ACID) |
| PageRank | Eventual | Updated periodically (not real-time) |
| Query logs | Eventual | Processed in batches |

**Eventual Consistency Boundaries:**
- Index staleness: Up to hours/days (acceptable for web search)
- New pages: Indexed within crawl cycle (hours to days)
- PageRank: Updated periodically (not real-time)
- No read-after-write needed for search (read-only operation)

---

## Inverted Index Structure

The inverted index is the core data structure. It's stored in a custom binary format optimized for fast access.

### What is an Inverted Index?

An inverted index maps terms to the documents containing them. It's "inverted" because instead of document → terms, we store term → documents.

```
Forward Index (Document-centric):
  Doc1 → [the, quick, brown, fox]
  Doc2 → [the, lazy, dog]

Inverted Index (Term-centric):
  the   → [Doc1, Doc2]
  quick → [Doc1]
  brown → [Doc1]
  fox   → [Doc1]
  lazy  → [Doc2]
  dog   → [Doc2]
```

### Conceptual Structure

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

### Index Implementation

```java
public class InvertedIndex {
    
    // In-memory term dictionary
    private Map<String, TermEntry> termDictionary;
    
    // Memory-mapped posting lists on disk
    private MappedByteBuffer postingLists;
    
    public static class TermEntry {
        String term;
        int documentFrequency;  // Number of docs containing term
        long postingListOffset; // Offset in posting file
        int postingListLength;  // Length in bytes
    }
    
    public static class Posting {
        int docId;
        short termFrequency;    // Times term appears in doc
        int[] positions;        // Word positions for phrase queries
        byte fieldFlags;        // Which fields contain term (title, body, etc.)
    }
}
```

### Compression Techniques

**Delta Encoding:**

```
Original doc IDs:  [100, 150, 152, 200, 201, 205]
Delta encoded:     [100,  50,   2,  48,   1,   4]
```

**Variable Byte Encoding (VByte):**

```java
public class VByteEncoder {
    
    public byte[] encode(int[] numbers) {
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        
        for (int num : numbers) {
            while (num >= 128) {
                out.write((num & 0x7F) | 0x80);  // Set continuation bit
                num >>>= 7;
            }
            out.write(num);  // Last byte, no continuation bit
        }
        
        return out.toByteArray();
    }
}
```

**Compression Ratios:**

| Technique | Bits per posting | Compression ratio |
|-----------|------------------|-------------------|
| Uncompressed | 32 | 1x |
| Delta + VByte | 8-12 | 3-4x |
| PForDelta | 4-8 | 4-8x |
| SIMD-BP128 | 4-6 | 5-8x |

---

## Document Metadata Table (PostgreSQL)

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

## URL Frontier Table (PostgreSQL + Redis)

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

## Link Graph Table

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

## Query Logs Table (ClickHouse)

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

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    USERS                                             │
│                         (Web Browsers, Mobile Apps, API)                            │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                               QUERY PATH (Online)                                    │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │    CDN      │───>│    Load     │───>│   Query     │───>│   Results   │          │
│  │  (Cache)    │    │  Balancer   │    │ Coordinator │    │   Cache     │          │
│  └─────────────┘    └─────────────┘    └──────┬──────┘    └─────────────┘          │
│                                               │                                      │
│                     ┌─────────────────────────┼─────────────────────────┐           │
│                     ▼                         ▼                         ▼           │
│              ┌─────────────┐          ┌─────────────┐          ┌─────────────┐     │
│              │Index Shard 1│          │Index Shard 2│          │Index Shard N│     │
│              │  (Replica)  │          │  (Replica)  │          │  (Replica)  │     │
│              └─────────────┘          └─────────────┘          └─────────────┘     │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              INDEXING PATH (Offline)                                 │
│                                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │   Web       │───>│  Document   │───>│   Indexer   │───>│   Index     │          │
│  │  Crawler    │    │  Processor  │    │             │    │   Builder   │          │
│  └──────┬──────┘    └─────────────┘    └─────────────┘    └──────┬──────┘          │
│         │                                                         │                 │
│         ▼                                                         ▼                 │
│  ┌─────────────┐                                          ┌─────────────┐          │
│  │    URL      │                                          │   Index     │          │
│  │  Frontier   │                                          │   Shards    │          │
│  └─────────────┘                                          └─────────────┘          │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            OFFLINE PROCESSING                                        │
│                                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                             │
│  │  PageRank   │    │   Spam      │    │   Quality   │                             │
│  │  Computer   │    │  Detector   │    │   Scorer    │                             │
│  │  (MapReduce)│    │             │    │             │                             │
│  └─────────────┘    └─────────────┘    └─────────────┘                             │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Detailed Crawler Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              WEB CRAWLER SYSTEM                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

                              ┌───────────────────┐
                              │   Seed URLs       │
                              │   (Initial Set)   │
                              └─────────┬─────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              URL FRONTIER                                            │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         PRIORITY QUEUES                                      │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │    │
│  │  │ Priority 1  │  │ Priority 2  │  │ Priority 3  │  │ Priority N  │        │    │
│  │  │ (News)      │  │ (Popular)   │  │ (Normal)    │  │ (Low)       │        │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         DOMAIN QUEUES (Politeness)                           │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │    │
│  │  │example.com  │  │ google.com  │  │  cnn.com    │  │   ...       │        │    │
│  │  │ [url1,url2] │  │ [url3,url4] │  │ [url5,url6] │  │             │        │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                     ┌──────────────────┼──────────────────┐
                     ▼                  ▼                  ▼
          ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
          │  Crawler Pod 1  │  │  Crawler Pod 2  │  │  Crawler Pod N  │
          │                 │  │                 │  │                 │
          │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │
          │ │ DNS Resolver│ │  │ │ DNS Resolver│ │  │ │ DNS Resolver│ │
          │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │
          │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │
          │ │ HTTP Client │ │  │ │ HTTP Client │ │  │ │ HTTP Client │ │
          │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │
          │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │
          │ │robots.txt   │ │  │ │robots.txt   │ │  │ │robots.txt   │ │
          │ │   Cache     │ │  │ │   Cache     │ │  │ │   Cache     │ │
          │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │
          └────────┬────────┘  └────────┬────────┘  └────────┬────────┘
                   │                    │                    │
                   └────────────────────┼────────────────────┘
                                        │
                                        ▼
                            ┌───────────────────────┐
                            │   Downloaded Pages    │
                            │   (Kafka Topic)       │
                            └───────────┬───────────┘
                                        │
                                        ▼
                            ┌───────────────────────┐
                            │   Document Store      │
                            │   (S3 / HDFS)         │
                            └───────────────────────┘
```

---

## Query Processing Flow

### Sequence Diagram

```
┌──────┐     ┌─────┐     ┌────────────┐     ┌────────────┐     ┌────────────┐     ┌────────────┐
│Client│     │ CDN │     │   Query    │     │   Cache    │     │   Index    │     │  Ranking   │
│      │     │     │     │Coordinator │     │  (Redis)   │     │  Shards    │     │  Engine    │
└──┬───┘     └──┬──┘     └─────┬──────┘     └─────┬──────┘     └─────┬──────┘     └─────┬──────┘
   │            │              │                  │                  │                  │
   │ GET /search?q=pizza      │                  │                  │                  │
   │───────────>│              │                  │                  │                  │
   │            │              │                  │                  │                  │
   │            │ Cache MISS   │                  │                  │                  │
   │            │─────────────>│                  │                  │                  │
   │            │              │                  │                  │                  │
   │            │              │ Check cache      │                  │                  │
   │            │              │─────────────────>│                  │                  │
   │            │              │                  │                  │                  │
   │            │              │     MISS         │                  │                  │
   │            │              │<─────────────────│                  │                  │
   │            │              │                  │                  │                  │
   │            │              │ Parse & expand query                │                  │
   │            │              │──────────────────────────────────────────────────────>│
   │            │              │                  │                  │                  │
   │            │              │ Broadcast to all shards             │                  │
   │            │              │─────────────────────────────────────>│                  │
   │            │              │                  │                  │                  │
   │            │              │              Top K results per shard│                  │
   │            │              │<─────────────────────────────────────│                  │
   │            │              │                  │                  │                  │
   │            │              │ Merge & re-rank  │                  │                  │
   │            │              │──────────────────────────────────────────────────────>│
   │            │              │                  │                  │                  │
   │            │              │              Final ranked results   │                  │
   │            │              │<──────────────────────────────────────────────────────│
   │            │              │                  │                  │                  │
   │            │              │ Cache results    │                  │                  │
   │            │              │─────────────────>│                  │                  │
   │            │              │                  │                  │                  │
   │            │ Results      │                  │                  │                  │
   │            │<─────────────│                  │                  │                  │
   │            │              │                  │                  │                  │
   │ Results    │              │                  │                  │                  │
   │<───────────│              │                  │                  │                  │
```

### Step-by-Step Explanation

1. **User submits query** (e.g., "best pizza NYC")
2. **CDN checks cache** for exact query match
3. **Query Coordinator** receives request
4. **Check Redis cache** for cached results
5. **Parse query**: tokenize, remove stop words, expand synonyms
6. **Spell check**: suggest corrections if needed
7. **Broadcast to shards**: query all index shards in parallel
8. **Each shard returns**: top K documents with scores
9. **Merge results**: combine and re-rank globally
10. **Apply personalization**: adjust for user location/language
11. **Cache results**: store in Redis for future queries
12. **Return to user**: formatted search results

---

## Indexing Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              INDEXING PIPELINE                                       │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐
│  Raw HTML       │
│  (from Crawler) │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 1: DOCUMENT PROCESSING                                                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                      │
│  │ HTML Parser     │  │ Content         │  │ Metadata        │                      │
│  │                 │  │ Extractor       │  │ Extractor       │                      │
│  │ - Parse DOM     │  │ - Remove ads    │  │ - Title         │                      │
│  │ - Handle errors │  │ - Remove nav    │  │ - Description   │                      │
│  │ - Detect charset│  │ - Main content  │  │ - Author        │                      │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 2: TEXT PROCESSING                                                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                      │
│  │ Tokenizer       │  │ Normalizer      │  │ Stemmer         │                      │
│  │                 │  │                 │  │                 │                      │
│  │ - Word boundary │  │ - Lowercase     │  │ - Porter stem   │                      │
│  │ - Punctuation   │  │ - Unicode norm  │  │ - Lemmatization │                      │
│  │ - Numbers       │  │ - Accents       │  │                 │                      │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                      │
│                                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐                                           │
│  │ Stop Word       │  │ Language        │                                           │
│  │ Removal         │  │ Detection       │                                           │
│  │                 │  │                 │                                           │
│  │ - the, a, is    │  │ - n-gram based  │                                           │
│  │ - Common words  │  │ - 50+ languages │                                           │
│  └─────────────────┘  └─────────────────┘                                           │
└─────────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 3: INDEX BUILDING                                                             │
│                                                                                      │
│  Document: "Best pizza in New York City"                                            │
│  Tokens: [best, pizza, new, york, city]                                             │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         INVERTED INDEX UPDATE                                │    │
│  │                                                                              │    │
│  │  Term: "pizza"                                                               │    │
│  │  ┌──────────────────────────────────────────────────────────────────────┐   │    │
│  │  │ Posting List:                                                         │   │    │
│  │  │ doc_123: {tf: 1, positions: [2], fields: [title]}                    │   │    │
│  │  │ doc_456: {tf: 3, positions: [5, 12, 89], fields: [body]}             │   │    │
│  │  │ doc_789: {tf: 2, positions: [1, 45], fields: [title, body]}          │   │    │
│  │  │ ... (millions more)                                                   │   │    │
│  │  └──────────────────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 4: INDEX DISTRIBUTION                                                         │
│                                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │  Shard 0    │    │  Shard 1    │    │  Shard 2    │    │  Shard N    │          │
│  │  Docs 0-99M │    │ Docs 100-   │    │ Docs 200-   │    │    ...      │          │
│  │             │    │    199M     │    │    299M     │    │             │          │
│  │ ┌─────────┐ │    │ ┌─────────┐ │    │ ┌─────────┐ │    │ ┌─────────┐ │          │
│  │ │Replica 1│ │    │ │Replica 1│ │    │ │Replica 1│ │    │ │Replica 1│ │          │
│  │ │Replica 2│ │    │ │Replica 2│ │    │ │Replica 2│ │    │ │Replica 2│ │          │
│  │ │Replica 3│ │    │ │Replica 3│ │    │ │Replica 3│ │    │ │Replica 3│ │          │
│  │ └─────────┘ │    │ └─────────┘ │    │ └─────────┘ │    │ └─────────┘ │          │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘          │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## PageRank Computation (MapReduce)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              PAGERANK COMPUTATION                                    │
└─────────────────────────────────────────────────────────────────────────────────────┘

ITERATION N:

┌─────────────────────────────────────────────────────────────────────────────────────┐
│  MAP PHASE                                                                           │
│                                                                                      │
│  Input: (page_id, {current_rank, outlinks[]})                                       │
│                                                                                      │
│  For each page:                                                                      │
│    contribution = current_rank / num_outlinks                                        │
│    For each outlink:                                                                 │
│      emit(outlink_id, contribution)                                                  │
│                                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                             │
│  │  Mapper 1   │    │  Mapper 2   │    │  Mapper N   │                             │
│  │             │    │             │    │             │                             │
│  │ Page A: 0.5 │    │ Page D: 0.3 │    │ Page G: 0.2 │                             │
│  │ Links: B,C  │    │ Links: E,F  │    │ Links: H    │                             │
│  │             │    │             │    │             │                             │
│  │ Emit:       │    │ Emit:       │    │ Emit:       │                             │
│  │ (B, 0.25)   │    │ (E, 0.15)   │    │ (H, 0.2)    │                             │
│  │ (C, 0.25)   │    │ (F, 0.15)   │    │             │                             │
│  └─────────────┘    └─────────────┘    └─────────────┘                             │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  REDUCE PHASE                                                                        │
│                                                                                      │
│  For each page:                                                                      │
│    new_rank = (1 - d) / N + d * sum(contributions)                                  │
│    where d = 0.85 (damping factor), N = total pages                                 │
│                                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                             │
│  │  Reducer 1  │    │  Reducer 2  │    │  Reducer N  │                             │
│  │             │    │             │    │             │                             │
│  │ Page B:     │    │ Page E:     │    │ Page H:     │                             │
│  │ Inputs:     │    │ Inputs:     │    │ Inputs:     │                             │
│  │ 0.25, 0.1,  │    │ 0.15, 0.3   │    │ 0.2, 0.1    │                             │
│  │ 0.05        │    │             │    │             │                             │
│  │             │    │             │    │             │                             │
│  │ new_rank =  │    │ new_rank =  │    │ new_rank =  │                             │
│  │ 0.355       │    │ 0.395       │    │ 0.27        │                             │
│  └─────────────┘    └─────────────┘    └─────────────┘                             │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ Repeat until convergence
                                        │ (typically 20-50 iterations)
                                        ▼
                            ┌───────────────────────┐
                            │   Final PageRank      │
                            │   Scores              │
                            └───────────────────────┘
```

---

## Caching Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              MULTI-LAYER CACHING                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘

                               ┌───────────────────┐
                               │      USER         │
                               └─────────┬─────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 1: CDN CACHE (Edge)                                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  Cache Key: hash(query + location + language)                                │    │
│  │  TTL: 5 minutes (popular queries)                                            │    │
│  │  Hit Rate: ~30% for popular queries                                          │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         │ Cache MISS
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 2: RESULTS CACHE (Redis Cluster)                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  Cache Key: hash(normalized_query)                                           │    │
│  │  Value: Serialized top 100 results                                           │    │
│  │  TTL: 1 hour                                                                 │    │
│  │  Size: 40 GB (10M queries × 4 KB)                                            │    │
│  │  Hit Rate: ~60%                                                              │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         │ Cache MISS
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 3: POSTING LIST CACHE (In-Memory)                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  Top 100K terms (by frequency) cached in memory                              │    │
│  │  Size: 10 TB distributed across index servers                                │    │
│  │  Hit Rate: ~95% of term lookups                                              │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         │ Cache MISS
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 4: DISK (SSD)                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  Full inverted index on NVMe SSDs                                            │    │
│  │  Memory-mapped for fast random access                                        │    │
│  │  Size: 120 TB                                                                │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Failure Points and Mitigation

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              FAILURE ANALYSIS                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

Component              Failure Mode           Impact              Mitigation
─────────────────────────────────────────────────────────────────────────────────────

┌─────────────────┐
│ Query           │ ─── Coordinator ───── Queries fail ───── Multiple coordinators
│ Coordinator     │     crash                                  behind load balancer
└─────────────────┘

┌─────────────────┐
│ Index Shard     │ ─── Shard down ─────── Partial results ── 3x replication,
│                 │                        (degraded)          query other replicas
└─────────────────┘

┌─────────────────┐
│ Results Cache   │ ─── Redis node ─────── Higher latency ─── Redis Cluster with
│ (Redis)         │     failure            (cache miss)        automatic failover
└─────────────────┘

┌─────────────────┐
│ Crawler         │ ─── Crawler pod ────── Crawl slows ────── Auto-restart,
│                 │     crash              down                 HPA scaling
└─────────────────┘

┌─────────────────┐
│ URL Frontier    │ ─── Queue loss ─────── URLs lost ──────── Kafka persistence,
│                 │                                            Redis AOF
└─────────────────┘

┌─────────────────┐
│ Document Store  │ ─── S3/HDFS ────────── Index rebuild ──── Multi-AZ replication,
│                 │     failure            needed               cross-region backup
└─────────────────┘

┌─────────────────┐
│ PageRank Job    │ ─── MapReduce ──────── Stale scores ───── Checkpointing,
│                 │     failure                                 job retry
└─────────────────┘
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

## Summary

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Query path | Scatter-gather | Parallel shard queries |
| Index storage | Custom + SSD | Optimized for posting list access |
| Caching | 4-layer (CDN→Redis→Mem→SSD) | Progressive latency reduction |
| Crawler | Distributed pods | Scalable, politeness per domain |
| PageRank | MapReduce (Spark) | Handles graph at scale |
| Replication | 3x per shard | Fault tolerance + read scaling |
| Deployment | Kubernetes | Auto-scaling, self-healing |

