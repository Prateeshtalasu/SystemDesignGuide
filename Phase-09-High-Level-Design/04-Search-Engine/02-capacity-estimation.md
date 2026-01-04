# Search Engine - Capacity Estimation

## Overview

This document calculates the infrastructure requirements for a web search engine handling 10 billion pages and 100,000 queries per second.

---

## Traffic Estimation

### Query Traffic

**Given:**
- Peak QPS: 100,000 queries/second
- Average QPS: 50,000 queries/second (assuming 2x peak factor)

**Daily Queries:**
```
Average daily = 50,000 × 86,400 seconds
             = 4.32 billion queries/day

Peak daily   = 100,000 × 86,400 seconds
             = 8.64 billion queries/day
```

**Monthly Queries:**
```
Monthly queries = 4.32B × 30 days
               = ~130 billion queries/month
```

### Crawl Traffic

**Given:**
- 10 billion pages to index
- Re-crawl cycle: 30 days average
- News pages: re-crawl every hour

**Crawl Rate:**
```
Base crawl rate = 10B pages / 30 days
               = 333 million pages/day
               = 3,858 pages/second

News crawl (1M pages hourly) = 1,000,000 / 3,600
                             = 278 pages/second

Total crawl rate = ~4,000 pages/second
```

---

## Storage Estimation

### Raw Document Storage

**Average Page Size:**
- Raw HTML: 100 KB average
- After compression (gzip): ~20 KB
- Text content only: ~5 KB

**Total Raw Storage:**
```
Compressed HTML = 10B pages × 20 KB
               = 200 TB

Text content only = 10B pages × 5 KB
                 = 50 TB
```

**Math Verification:**
- Assumptions: 10B pages, 20 KB compressed HTML per page, 5 KB text per page
- Compressed HTML: 10,000,000,000 × 20 KB = 200,000,000,000 KB = 200 TB
- Text content: 10,000,000,000 × 5 KB = 50,000,000,000 KB = 50 TB
- **DOC MATCHES:** Storage summary shows 200 TB + 50 TB = 250 TB ✅

### Inverted Index Storage

**Index Structure:**
```
For each term:
- Term ID: 4 bytes
- Document frequency: 4 bytes
- Posting list: variable (doc_id + position + score)
```

**Posting List Size:**
```
Average term appears in: 1% of documents = 100M documents
Each posting: doc_id (4B) + position (4B) + score (4B) = 12 bytes

Average posting list = 100M × 12 bytes = 1.2 GB per term
```

**Total Index Size:**
```
Unique terms (vocabulary): ~10 million terms
Average posting list: varies widely

Compressed index estimate:
- High-frequency terms (10K): 1.2 GB × 10K = 12 TB
- Medium-frequency terms (1M): 100 MB × 1M = 100 TB
- Low-frequency terms (9M): 1 MB × 9M = 9 TB

Total inverted index ≈ 120 TB (compressed)
```

**Math Verification:**
- Assumptions: 10M unique terms, posting lists vary by frequency
- High-frequency (10K terms): 10,000 × 1.2 GB = 12 TB
- Medium-frequency (1M terms): 1,000,000 × 100 MB = 100 TB
- Low-frequency (9M terms): 9,000,000 × 1 MB = 9 TB
- Total: 12 + 100 + 9 = 121 TB ≈ 120 TB (rounded)
- **DOC MATCHES:** Storage summary shows 120 TB for inverted index ✅

### Document Metadata Storage

**Per Document:**
```
- Document ID: 8 bytes
- URL: 100 bytes average
- Title: 60 bytes average
- Snippet: 200 bytes average
- PageRank score: 4 bytes
- Timestamp: 8 bytes
- Other metadata: 50 bytes

Total per document: ~430 bytes
```

**Total Metadata:**
```
10B documents × 430 bytes = 4.3 TB
```

**Math Verification:**
- Assumptions: 10B documents, 430 bytes per document
- Calculation: 10,000,000,000 × 430 bytes = 4,300,000,000,000 bytes = 4.3 TB
- **DOC MATCHES:** Storage summary shows 4.3 TB for metadata ✅

### URL Frontier Storage (Crawler)

**URLs to Crawl:**
```
- Active frontier: 1 billion URLs
- Each URL: 100 bytes average

Frontier storage = 1B × 100 bytes = 100 GB
```

### Total Storage Summary

| Component           | Size       | Notes                           |
| ------------------- | ---------- | ------------------------------- |
| Compressed HTML     | 200 TB     | Raw crawled pages               |
| Text content        | 50 TB      | Extracted text                  |
| Inverted index      | 120 TB     | Compressed posting lists        |
| Document metadata   | 4.3 TB     | Titles, URLs, snippets          |
| URL frontier        | 100 GB     | Crawler queue                   |
| **Total**           | **~375 TB**| Plus replication = ~1 PB        |

**With 3x Replication:**
```
Total storage = 375 TB × 3 = 1.125 PB
```

---

## Bandwidth Estimation

### Query Bandwidth

**Request Size:**
```
- Query string: 50 bytes average
- Headers: 200 bytes
- Total request: ~250 bytes
```

**Response Size:**
```
- 10 results per page
- Each result: title (60B) + URL (100B) + snippet (200B) = 360 bytes
- 10 results: 3,600 bytes
- Headers + metadata: 400 bytes
- Total response: ~4 KB
```

**Query Bandwidth:**
```
Incoming = 100K QPS × 250 bytes = 25 MB/s
Outgoing = 100K QPS × 4 KB = 400 MB/s = 3.2 Gbps
```

### Crawl Bandwidth

**Crawl Traffic:**
```
Pages downloaded = 4,000 pages/second × 100 KB = 400 MB/s
                 = 3.2 Gbps incoming
```

### Total Bandwidth

| Direction  | Traffic   | Notes                    |
| ---------- | --------- | ------------------------ |
| Query in   | 25 MB/s   | User queries             |
| Query out  | 400 MB/s  | Search results           |
| Crawl in   | 400 MB/s  | Downloaded pages         |
| **Total**  | ~825 MB/s | ~6.6 Gbps                |

---

## Memory Estimation

### Query Processing Memory

**Hot Index in Memory:**
```
- Top 1% of terms (100K terms) = 90% of queries
- These posting lists: ~10 TB
- Keep in distributed memory across cluster
```

**Per Query Memory:**
```
- Query parsing: 1 KB
- Intermediate results: 100 KB
- Final results: 10 KB

Memory per query = ~111 KB
```

**Concurrent Queries:**
```
At 100K QPS with 100ms latency:
Concurrent queries = 100K × 0.1 = 10,000 queries

Memory for concurrent queries = 10K × 111 KB = 1.1 GB per server
```

### Cache Memory

**Query Result Cache:**
```
- Cache top 10 million queries
- Each cached result: 4 KB
- Cache size = 10M × 4 KB = 40 GB

Distributed across 10 cache servers = 4 GB each
```

**Document Cache:**
```
- Cache top 100 million document metadata
- Each document: 430 bytes
- Cache size = 100M × 430 bytes = 43 GB
```

### Total Memory Requirements

| Component              | Memory    | Notes                      |
| ---------------------- | --------- | -------------------------- |
| Hot index              | 10 TB     | Distributed across cluster |
| Query result cache     | 40 GB     | Top queries                |
| Document cache         | 43 GB     | Metadata                   |
| Query processing       | 1.1 GB    | Per server                 |

---

## Server Estimation

### Query Servers

**Assumptions:**
- Each server handles 5,000 QPS
- Each query needs to access 10 shards in parallel

**Query Servers Needed:**
```
Query servers = 100K QPS / 5K QPS per server
             = 20 query coordinator servers
```

### Index Servers (Shards)

**Index Sharding:**
```
Total index size: 120 TB
Memory per server: 256 GB (for hot data)
Disk per server: 4 TB SSD

Servers for hot index = 10 TB / 256 GB = 40 servers
Servers for full index = 120 TB / 4 TB = 30 servers

With replication (3x) = 30 × 3 = 90 index servers
```

**Index Shard Configuration:**
```
- 100 shards (each ~1.2 TB)
- 3 replicas per shard
- 300 shard replicas total
- 3-4 shards per server
```

### Crawler Servers

**Crawl Requirements:**
```
Crawl rate: 4,000 pages/second
Pages per crawler: 100 pages/second (with politeness delays)

Crawler servers = 4,000 / 100 = 40 crawlers
```

### Document Processing Servers

**Processing Rate:**
```
Pages to process: 4,000/second
Processing time: 500ms per page
Pages per server: 50/second

Processing servers = 4,000 / 50 = 80 servers
```

### Server Summary

| Component           | Servers | Specs                           |
| ------------------- | ------- | ------------------------------- |
| Query coordinators  | 20      | 32 cores, 64 GB RAM             |
| Index servers       | 90      | 32 cores, 256 GB RAM, 4 TB SSD  |
| Crawlers            | 40      | 16 cores, 32 GB RAM             |
| Doc processors      | 80      | 32 cores, 64 GB RAM             |
| Cache servers       | 10      | 16 cores, 128 GB RAM            |
| **Total**           | **240** | Plus management/monitoring      |

---

## Cost Estimation

### Compute Costs (AWS)

| Component           | Instance Type | Count | Monthly Cost |
| ------------------- | ------------- | ----- | ------------ |
| Query coordinators  | c6i.8xlarge   | 20    | $20,000      |
| Index servers       | r6i.8xlarge   | 90    | $135,000     |
| Crawlers            | c6i.4xlarge   | 40    | $24,000      |
| Doc processors      | c6i.8xlarge   | 80    | $80,000      |
| Cache servers       | r6i.4xlarge   | 10    | $7,500       |
| **Compute Total**   |               |       | **$266,500** |

### Storage Costs

| Type                | Size    | Monthly Cost |
| ------------------- | ------- | ------------ |
| SSD (index)         | 400 TB  | $40,000      |
| HDD (raw pages)     | 600 TB  | $12,000      |
| S3 (backup)         | 1 PB    | $23,000      |
| **Storage Total**   |         | **$75,000**  |

### Network Costs

```
Outbound bandwidth: 400 MB/s = ~1 PB/month
At $0.05/GB = $50,000/month
```

### Total Monthly Cost

| Category    | Cost         |
| ----------- | ------------ |
| Compute     | $266,500     |
| Storage     | $75,000      |
| Network     | $50,000      |
| Misc (20%)  | $78,300      |
| **Total**   | **~$470,000/month** |

### Cost per Query

```
Monthly queries: 130 billion
Monthly cost: $470,000

Cost per query = $470,000 / 130B = $0.0000036
              = $3.60 per million queries
```

---

## Scaling Projections

### 1-Year Growth (2x)

| Metric          | Current     | 1 Year      |
| --------------- | ----------- | ----------- |
| Pages indexed   | 10B         | 20B         |
| QPS             | 100K        | 200K        |
| Index size      | 120 TB      | 240 TB      |
| Servers         | 240         | 480         |
| Monthly cost    | $470K       | $940K       |

### 5-Year Growth (10x)

| Metric          | Current     | 5 Years     |
| --------------- | ----------- | ----------- |
| Pages indexed   | 10B         | 100B        |
| QPS             | 100K        | 1M          |
| Index size      | 120 TB      | 1.2 PB      |
| Servers         | 240         | 2,400       |
| Monthly cost    | $470K       | $4.7M       |

---

## Quick Reference Numbers

### Interview Mental Math

```
10B pages indexed
100K QPS
120 TB index size
240 servers
$470K/month
$3.60 per million queries
```

### Key Ratios

```
Index size per page: ~12 KB (compressed)
Queries per server: 5,000 QPS
Pages per crawler: 100/second
Memory per query: ~100 KB
```

---

## Summary

| Aspect              | Value                                  |
| ------------------- | -------------------------------------- |
| Query QPS           | 100,000 peak                           |
| Pages indexed       | 10 billion                             |
| Index size          | 120 TB (compressed)                    |
| Total storage       | ~1 PB (with replication)               |
| Query bandwidth     | 400 MB/s outbound                      |
| Index servers       | 90 (with 3x replication)               |
| Total servers       | ~240                                   |
| Monthly cost        | ~$470,000                              |
| Cost per query      | $0.0000036                             |

