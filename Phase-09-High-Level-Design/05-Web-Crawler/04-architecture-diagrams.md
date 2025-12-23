# Web Crawler - Architecture Diagrams

## Component Overview

Before looking at diagrams, let's understand each component and why it exists.

### Components Explained

| Component              | Purpose                                | Why It Exists                                    |
| ---------------------- | -------------------------------------- | ------------------------------------------------ |
| **URL Frontier**       | Queue of URLs to crawl                 | Manages what to crawl next                       |
| **Scheduler**          | Decides when to crawl each URL         | Enforces politeness and priorities               |
| **Fetcher**            | Downloads web pages                    | Core crawling functionality                      |
| **DNS Resolver**       | Resolves domain names                  | Must know IP to connect                          |
| **Parser**             | Extracts links and content             | Discovers new URLs                               |
| **Duplicate Detector** | Filters seen URLs/content              | Avoids wasting resources                         |
| **Robots Handler**     | Manages robots.txt rules               | Legal and ethical compliance                     |
| **Document Store**     | Stores crawled pages                   | Persistence for indexing                         |
| **Coordinator**        | Manages distributed crawlers           | Scales across multiple machines                  |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              WEB CRAWLER SYSTEM                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

                              ┌───────────────────┐
                              │    Seed URLs      │
                              │   (Initial Set)   │
                              └─────────┬─────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              COORDINATOR CLUSTER                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         URL FRONTIER                                         │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │    │
│  │  │ Priority Q 1 │  │ Priority Q 2 │  │ Priority Q 3 │  │ Priority Q N │    │    │
│  │  │ (Urgent)     │  │ (High)       │  │ (Normal)     │  │ (Low)        │    │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         SCHEDULER                                            │    │
│  │  - Politeness enforcement (per-domain delays)                                │    │
│  │  - Priority-based selection                                                  │    │
│  │  - Domain queue management                                                   │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                     ┌──────────────────┼──────────────────┐
                     ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              CRAWLER WORKERS                                         │
│                                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                     │
│  │  Crawler Pod 1  │  │  Crawler Pod 2  │  │  Crawler Pod N  │                     │
│  │                 │  │                 │  │                 │                     │
│  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │                     │
│  │ │   Fetcher   │ │  │ │   Fetcher   │ │  │ │   Fetcher   │ │                     │
│  │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │                     │
│  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │                     │
│  │ │   Parser    │ │  │ │   Parser    │ │  │ │   Parser    │ │                     │
│  │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │                     │
│  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │                     │
│  │ │ DNS Cache   │ │  │ │ DNS Cache   │ │  │ │ DNS Cache   │ │                     │
│  │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │                     │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘                     │
│           │                    │                    │                              │
└───────────┼────────────────────┼────────────────────┼──────────────────────────────┘
            │                    │                    │
            └────────────────────┼────────────────────┘
                                 │
                     ┌───────────┴───────────┐
                     ▼                       ▼
          ┌─────────────────┐     ┌─────────────────┐
          │  Document Store │     │  Kafka Topics   │
          │  (S3 / HDFS)    │     │  (New URLs)     │
          └─────────────────┘     └─────────────────┘
```

---

## Detailed Crawler Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              SINGLE CRAWLER POD                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         URL INPUT QUEUE                                      │    │
│  │                    (From Kafka / Coordinator)                                │    │
│  └────────────────────────────────┬────────────────────────────────────────────┘    │
│                                   │                                                  │
│                                   ▼                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         ROBOTS.TXT CHECKER                                   │    │
│  │                                                                              │    │
│  │  ┌─────────────────┐    ┌─────────────────┐                                 │    │
│  │  │ Robots Cache    │    │ Robots Fetcher  │                                 │    │
│  │  │ (Redis)         │    │ (if not cached) │                                 │    │
│  │  └─────────────────┘    └─────────────────┘                                 │    │
│  │                                                                              │    │
│  │  Decision: ALLOW / DISALLOW / CRAWL_DELAY                                   │    │
│  └────────────────────────────────┬────────────────────────────────────────────┘    │
│                                   │                                                  │
│                                   ▼                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         DNS RESOLVER                                         │    │
│  │                                                                              │    │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐          │    │
│  │  │ Local DNS Cache │───>│ DNS Client      │───>│ External DNS    │          │    │
│  │  │ (In-Memory)     │    │ (Async)         │    │ (8.8.8.8)       │          │    │
│  │  └─────────────────┘    └─────────────────┘    └─────────────────┘          │    │
│  └────────────────────────────────┬────────────────────────────────────────────┘    │
│                                   │                                                  │
│                                   ▼                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         HTTP FETCHER                                         │    │
│  │                                                                              │    │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐          │    │
│  │  │ Connection Pool │    │ HTTP Client     │    │ Response        │          │    │
│  │  │ (per domain)    │    │ (Async/NIO)     │    │ Handler         │          │    │
│  │  │                 │    │                 │    │                 │          │    │
│  │  │ Keep-Alive      │    │ - Timeouts      │    │ - Decompress    │          │    │
│  │  │ Max 2 per host  │    │ - Redirects     │    │ - Validate      │          │    │
│  │  │                 │    │ - SSL/TLS       │    │ - Store         │          │    │
│  │  └─────────────────┘    └─────────────────┘    └─────────────────┘          │    │
│  └────────────────────────────────┬────────────────────────────────────────────┘    │
│                                   │                                                  │
│                                   ▼                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         HTML PARSER                                          │    │
│  │                                                                              │    │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐          │    │
│  │  │ DOM Parser      │    │ Link Extractor  │    │ Content         │          │    │
│  │  │ (JSoup)         │    │                 │    │ Extractor       │          │    │
│  │  │                 │    │ - <a href>      │    │                 │          │    │
│  │  │ - Handle errors │    │ - <link>        │    │ - Title         │          │    │
│  │  │ - Charset detect│    │ - <script src>  │    │ - Meta tags     │          │    │
│  │  │                 │    │ - Normalize     │    │ - Main content  │          │    │
│  │  └─────────────────┘    └─────────────────┘    └─────────────────┘          │    │
│  └────────────────────────────────┬────────────────────────────────────────────┘    │
│                                   │                                                  │
│                    ┌──────────────┼──────────────┐                                  │
│                    ▼              ▼              ▼                                  │
│           ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                       │
│           │ New URLs     │ │ Document     │ │ Metrics      │                       │
│           │ (to Kafka)   │ │ (to S3)      │ │ (to Prom)    │                       │
│           └──────────────┘ └──────────────┘ └──────────────┘                       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## URL Frontier Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              URL FRONTIER DETAIL                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘

                         ┌───────────────────────────┐
                         │      NEW URLS INPUT       │
                         │   (From Parsers/Seeds)    │
                         └─────────────┬─────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         DUPLICATE URL FILTER                                         │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         BLOOM FILTER                                         │    │
│  │                                                                              │    │
│  │  Size: 18 GB (10B URLs, 0.1% false positive)                                │    │
│  │                                                                              │    │
│  │  URL Hash → Bloom Filter → Seen? → Yes: Discard                             │    │
│  │                                  → No: Continue                              │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         URL NORMALIZER                                       │    │
│  │                                                                              │    │
│  │  - Lowercase host                                                           │    │
│  │  - Remove default ports                                                     │    │
│  │  - Sort query params                                                        │    │
│  │  - Remove tracking params                                                   │    │
│  │  - Remove fragments                                                         │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         PRIORITY ASSIGNMENT                                          │
│                                                                                      │
│  Priority Score = f(domain_importance, freshness_need, depth, page_type)            │
│                                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                     │
│  │ Domain Score    │  │ Freshness Score │  │ Depth Penalty   │                     │
│  │                 │  │                 │  │                 │                     │
│  │ PageRank-based  │  │ News: High      │  │ Depth 0: 1.0    │                     │
│  │ 0.0 - 1.0       │  │ Blog: Medium    │  │ Depth 1: 0.9    │                     │
│  │                 │  │ Static: Low     │  │ Depth 2: 0.8    │                     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         DOMAIN-BASED QUEUES                                          │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         BACK QUEUES (per domain)                             │    │
│  │                                                                              │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │    │
│  │  │ example.com  │  │ another.com  │  │  news.com    │  │    ...       │    │    │
│  │  │              │  │              │  │              │  │              │    │    │
│  │  │ [url1, url2, │  │ [url3, url4] │  │ [url5, url6, │  │              │    │    │
│  │  │  url7, ...]  │  │              │  │  url8, ...]  │  │              │    │    │
│  │  │              │  │              │  │              │  │              │    │    │
│  │  │ Last: 10:05  │  │ Last: 10:03  │  │ Last: 10:06  │  │              │    │    │
│  │  │ Delay: 1s    │  │ Delay: 2s    │  │ Delay: 0.5s  │  │              │    │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         FRONT QUEUES (by priority)                           │    │
│  │                                                                              │    │
│  │  Priority 1 (Urgent):  [domain_ptr1, domain_ptr2, ...]                      │    │
│  │  Priority 2 (High):    [domain_ptr3, domain_ptr4, ...]                      │    │
│  │  Priority 3 (Normal):  [domain_ptr5, domain_ptr6, ...]                      │    │
│  │  Priority 4 (Low):     [domain_ptr7, domain_ptr8, ...]                      │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                         ┌───────────────────────────┐
                         │     SCHEDULER OUTPUT      │
                         │    (URLs to Crawlers)     │
                         └───────────────────────────┘
```

---

## Crawl Flow Sequence

```
┌────────┐  ┌──────────┐  ┌────────┐  ┌───────┐  ┌────────┐  ┌────────┐  ┌───────┐
│Frontier│  │Coordinator│  │Crawler │  │ DNS   │  │ Web    │  │ Parser │  │ Store │
└───┬────┘  └────┬─────┘  └───┬────┘  └───┬───┘  └───┬────┘  └───┬────┘  └───┬───┘
    │            │            │           │          │           │           │
    │ Get next URL batch      │           │          │           │           │
    │<───────────│            │           │          │           │           │
    │            │            │           │          │           │           │
    │ URLs with priorities    │           │          │           │           │
    │───────────>│            │           │          │           │           │
    │            │            │           │          │           │           │
    │            │ Assign to crawler      │          │           │           │
    │            │───────────>│           │          │           │           │
    │            │            │           │          │           │           │
    │            │            │ Check robots.txt     │           │           │
    │            │            │──────────────────────────────────────────────>│
    │            │            │           │          │           │           │
    │            │            │ Resolve domain       │           │           │
    │            │            │──────────>│          │           │           │
    │            │            │           │          │           │           │
    │            │            │    IP     │          │           │           │
    │            │            │<──────────│          │           │           │
    │            │            │           │          │           │           │
    │            │            │ HTTP GET  │          │           │           │
    │            │            │─────────────────────>│           │           │
    │            │            │           │          │           │           │
    │            │            │         HTML         │           │           │
    │            │            │<─────────────────────│           │           │
    │            │            │           │          │           │           │
    │            │            │ Parse HTML│          │           │           │
    │            │            │──────────────────────────────────>│           │
    │            │            │           │          │           │           │
    │            │            │      Links + Content │           │           │
    │            │            │<──────────────────────────────────│           │
    │            │            │           │          │           │           │
    │            │            │ Store document       │           │           │
    │            │            │────────────────────────────────────────────>│
    │            │            │           │          │           │           │
    │ New URLs   │            │           │          │           │           │
    │<───────────────────────────────────────────────────────────│           │
    │            │            │           │          │           │           │
```

---

## Distributed Coordination

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         DISTRIBUTED CRAWLER COORDINATION                             │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         DOMAIN PARTITIONING                                          │
│                                                                                      │
│  Domain Hash → Partition Assignment                                                  │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                                                                              │    │
│  │   hash("example.com") % 15 = 3  →  Crawler Pod 3                            │    │
│  │   hash("another.com") % 15 = 7  →  Crawler Pod 7                            │    │
│  │   hash("news.com") % 15 = 12    →  Crawler Pod 12                           │    │
│  │                                                                              │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  Benefits:                                                                           │
│  - Same domain always goes to same crawler                                          │
│  - Politeness enforced locally (no coordination needed)                             │
│  - robots.txt cached locally                                                        │
│  - Connection reuse within pod                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         KAFKA-BASED URL DISTRIBUTION                                 │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         TOPIC: discovered-urls                               │    │
│  │                                                                              │    │
│  │  Partition 0 ──> Crawler 0                                                  │    │
│  │  Partition 1 ──> Crawler 1                                                  │    │
│  │  Partition 2 ──> Crawler 2                                                  │    │
│  │  ...                                                                        │    │
│  │  Partition 14 ──> Crawler 14                                                │    │
│  │                                                                              │    │
│  │  Key: domain_hash (ensures same domain → same partition)                    │    │
│  │  Value: {url, priority, depth, discovered_from}                             │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         TOPIC: crawled-documents                             │    │
│  │                                                                              │    │
│  │  All crawlers produce to this topic                                         │    │
│  │  Consumed by: Indexer, Analytics, Archive                                   │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         CRAWLER POD ASSIGNMENT                                       │
│                                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │ Crawler 0    │  │ Crawler 1    │  │ Crawler 2    │  │ Crawler N    │            │
│  │              │  │              │  │              │  │              │            │
│  │ Domains:     │  │ Domains:     │  │ Domains:     │  │ Domains:     │            │
│  │ - a.com      │  │ - b.com      │  │ - c.com      │  │ - z.com      │            │
│  │ - d.com      │  │ - e.com      │  │ - f.com      │  │ - ...        │            │
│  │ - g.com      │  │ - h.com      │  │ - i.com      │  │              │            │
│  │              │  │              │  │              │  │              │            │
│  │ Kafka:       │  │ Kafka:       │  │ Kafka:       │  │ Kafka:       │            │
│  │ Partition 0  │  │ Partition 1  │  │ Partition 2  │  │ Partition N  │            │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘            │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Failure Handling

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              FAILURE SCENARIOS                                       │
└─────────────────────────────────────────────────────────────────────────────────────┘

Failure Type           Detection              Recovery
─────────────────────────────────────────────────────────────────────────────────────

┌─────────────────┐
│ Crawler Pod     │ ─── K8s health ─────── Restart pod
│ Crash           │     check fails        Kafka rebalances partitions
│                 │                        URLs re-consumed from offset
└─────────────────┘

┌─────────────────┐
│ DNS Timeout     │ ─── 5 second ───────── Use cached IP if available
│                 │     timeout            Retry with backup DNS
│                 │                        Mark domain for later retry
└─────────────────┘

┌─────────────────┐
│ HTTP Timeout    │ ─── 30 second ──────── Retry up to 3 times
│                 │     timeout            Exponential backoff
│                 │                        Mark URL as failed after 3
└─────────────────┘

┌─────────────────┐
│ HTTP 5xx Error  │ ─── Status code ────── Retry with backoff
│                 │     500-599            Max 3 retries
│                 │                        Back off entire domain
└─────────────────┘

┌─────────────────┐
│ HTTP 429        │ ─── Rate limited ───── Honor Retry-After header
│ (Too Many Req)  │                        Increase domain delay
│                 │                        Back off for 1+ hours
└─────────────────┘

┌─────────────────┐
│ Connection      │ ─── TCP error ──────── Retry immediately
│ Reset           │                        Then exponential backoff
│                 │                        Check if site is down
└─────────────────┘

┌─────────────────┐
│ Kafka Broker    │ ─── Connection ─────── Automatic failover
│ Down            │     timeout            Producer retries
│                 │                        Consumer rebalances
└─────────────────┘

┌─────────────────┐
│ S3/Storage      │ ─── Write error ────── Retry with backoff
│ Failure         │                        Buffer locally
│                 │                        Alert if persists
└─────────────────┘
```

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         KUBERNETES DEPLOYMENT                                        │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                         CRAWLER DEPLOYMENT                                     │  │
│  │                                                                                │  │
│  │  apiVersion: apps/v1                                                          │  │
│  │  kind: Deployment                                                             │  │
│  │  spec:                                                                        │  │
│  │    replicas: 15                                                               │  │
│  │    template:                                                                  │  │
│  │      spec:                                                                    │  │
│  │        containers:                                                            │  │
│  │        - name: crawler                                                        │  │
│  │          resources:                                                           │  │
│  │            requests:                                                          │  │
│  │              cpu: "4"                                                         │  │
│  │              memory: "8Gi"                                                    │  │
│  │            limits:                                                            │  │
│  │              cpu: "8"                                                         │  │
│  │              memory: "16Gi"                                                   │  │
│  │          env:                                                                 │  │
│  │          - name: KAFKA_BROKERS                                                │  │
│  │            value: "kafka-0:9092,kafka-1:9092,kafka-2:9092"                    │  │
│  │          - name: DNS_SERVERS                                                  │  │
│  │            value: "10.0.0.10,10.0.0.11"                                       │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                         KAFKA STATEFULSET                                      │  │
│  │                                                                                │  │
│  │  replicas: 3                                                                  │  │
│  │  storage: 1TB SSD per broker                                                  │  │
│  │  topics:                                                                      │  │
│  │    - discovered-urls (partitions: 15, replication: 3)                        │  │
│  │    - crawled-documents (partitions: 10, replication: 3)                      │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                         REDIS CLUSTER                                          │  │
│  │                                                                                │  │
│  │  replicas: 6 (3 primary + 3 replica)                                         │  │
│  │  memory: 32GB per node                                                        │  │
│  │  usage: robots.txt cache, bloom filter, domain state                         │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                         EXTERNAL SERVICES                                      │  │
│  │                                                                                │  │
│  │  - AWS S3: Document storage                                                   │  │
│  │  - AWS RDS (PostgreSQL): Metadata, job state                                 │  │
│  │  - Prometheus + Grafana: Monitoring                                          │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

| Aspect            | Decision                | Rationale                               |
| ----------------- | ----------------------- | --------------------------------------- |
| Distribution      | Domain-based sharding   | Politeness enforced locally             |
| Coordination      | Kafka partitions        | Scalable, fault-tolerant                |
| URL Frontier      | Two-level queues        | Priority + politeness                   |
| Deduplication     | Bloom filter            | Memory-efficient for billions           |
| Storage           | S3 + PostgreSQL         | Cheap blob + structured metadata        |
| Failure handling  | Retry + backoff         | Graceful degradation                    |

