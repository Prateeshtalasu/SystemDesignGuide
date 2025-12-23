# Search Engine - Architecture Diagrams

## Component Overview

Before looking at diagrams, let's understand each component and why it exists.

### Components Explained

| Component              | Purpose                                | Why It Exists                                    |
| ---------------------- | -------------------------------------- | ------------------------------------------------ |
| **Web Crawler**        | Discovers and downloads web pages      | Can't index what we haven't fetched              |
| **URL Frontier**       | Queue of URLs to crawl                 | Manages crawl priority and scheduling            |
| **Document Processor** | Parses HTML, extracts content          | Raw HTML isn't searchable                        |
| **Indexer**            | Builds inverted index                  | Enables fast keyword lookup                      |
| **Index Shards**       | Distributed index storage              | Single machine can't hold 120 TB                 |
| **Query Processor**    | Parses and expands queries             | Transforms user input into searchable terms      |
| **Ranking Engine**     | Scores and orders results              | Relevance is key to user satisfaction            |
| **Results Cache**      | Caches popular query results           | Reduces latency for common queries               |
| **PageRank Computer**  | Calculates link-based authority        | Quality signal for ranking                       |

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
│  │                                                                              │    │
│  │  Term: "york"                                                                │    │
│  │  ┌──────────────────────────────────────────────────────────────────────┐   │    │
│  │  │ Posting List:                                                         │   │    │
│  │  │ doc_123: {tf: 1, positions: [4], fields: [title]}                    │   │    │
│  │  │ doc_234: {tf: 5, positions: [3, 15, 22, 67, 89], fields: [body]}     │   │    │
│  │  │ ...                                                                   │   │    │
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

## Deployment Architecture (Kubernetes)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              KUBERNETES CLUSTERS                                     │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                         QUERY CLUSTER (Latency-Sensitive)                      │  │
│  │                                                                                │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐               │  │
│  │  │query-coordinator│  │  index-server   │  │  cache-server   │               │  │
│  │  │  Deployment     │  │  StatefulSet    │  │  StatefulSet    │               │  │
│  │  │                 │  │                 │  │                 │               │  │
│  │  │ Replicas: 20    │  │ Replicas: 90    │  │ Replicas: 10    │               │  │
│  │  │ CPU: 32 cores   │  │ CPU: 32 cores   │  │ CPU: 16 cores   │               │  │
│  │  │ Memory: 64 GB   │  │ Memory: 256 GB  │  │ Memory: 128 GB  │               │  │
│  │  │ Disk: 100 GB    │  │ Disk: 4 TB SSD  │  │ Disk: 500 GB    │               │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘               │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                         CRAWL CLUSTER (Throughput-Optimized)                   │  │
│  │                                                                                │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐               │  │
│  │  │    crawler      │  │  doc-processor  │  │    indexer      │               │  │
│  │  │  Deployment     │  │  Deployment     │  │  Deployment     │               │  │
│  │  │                 │  │                 │  │                 │               │  │
│  │  │ Replicas: 40    │  │ Replicas: 80    │  │ Replicas: 20    │               │  │
│  │  │ CPU: 16 cores   │  │ CPU: 32 cores   │  │ CPU: 32 cores   │               │  │
│  │  │ Memory: 32 GB   │  │ Memory: 64 GB   │  │ Memory: 128 GB  │               │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘               │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                         BATCH CLUSTER (Spark/Hadoop)                           │  │
│  │                                                                                │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐               │  │
│  │  │  pagerank-job   │  │  spam-detector  │  │  quality-scorer │               │  │
│  │  │  (Spark)        │  │  (Spark ML)     │  │  (Spark)        │               │  │
│  │  │                 │  │                 │  │                 │               │  │
│  │  │ Executors: 100  │  │ Executors: 50   │  │ Executors: 30   │               │  │
│  │  │ Memory: 16 GB   │  │ Memory: 32 GB   │  │ Memory: 16 GB   │               │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘               │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

| Aspect            | Decision                | Rationale                               |
| ----------------- | ----------------------- | --------------------------------------- |
| Query path        | Scatter-gather          | Parallel shard queries                  |
| Index storage     | Custom + SSD            | Optimized for posting list access       |
| Caching           | 4-layer (CDN→Redis→Mem→SSD) | Progressive latency reduction       |
| Crawler           | Distributed pods        | Scalable, politeness per domain         |
| PageRank          | MapReduce (Spark)       | Handles graph at scale                  |
| Replication       | 3x per shard            | Fault tolerance + read scaling          |
| Deployment        | Kubernetes              | Auto-scaling, self-healing              |

