# Typeahead / Autocomplete - Architecture Diagrams

## Component Overview

| Component | Purpose | Why It Exists |
|-----------|---------|---------------|
| **CDN Edge** | Cache popular suggestions | Reduce latency for common queries |
| **Load Balancer** | Distribute traffic | Handle 500K QPS across servers |
| **Suggestion Service** | Serve suggestions | Core business logic, Trie lookup |
| **Trie Index** | In-memory prefix tree | O(prefix length) lookups |
| **Data Pipeline** | Build/update index | Process logs, build Trie |
| **Search Logs** | Raw query data | Source of truth for popularity |
| **Zookeeper** | Coordination | Index version management |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    CLIENTS                                           │
│                    (Web Browsers, Mobile Apps, Search Boxes)                         │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         │ GET /suggestions?q=how+to
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              CDN EDGE (CloudFront)                                   │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  Edge Cache                                                                  │   │
│  │  Key: /suggestions?q=how+to&limit=10                                        │   │
│  │  TTL: 5 minutes                                                              │   │
│  │  Hit Rate: ~40% (popular prefixes)                                           │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         │ Cache MISS
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            LOAD BALANCER (Layer 4)                                   │
│                                                                                      │
│  - Round-robin distribution                                                          │
│  - Health checks every 5 seconds                                                     │
│  - Connection draining for deployments                                               │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              ▼                          ▼                          ▼
┌─────────────────────────┐  ┌─────────────────────────┐  ┌─────────────────────────┐
│   Suggestion Service    │  │   Suggestion Service    │  │   Suggestion Service    │
│        (Pod 1)          │  │        (Pod 2)          │  │        (Pod N)          │
│                         │  │                         │  │                         │
│  ┌───────────────────┐  │  │  ┌───────────────────┐  │  │  ┌───────────────────┐  │
│  │   Trie Index      │  │  │  │   Trie Index      │  │  │  │   Trie Index      │  │
│  │   (In-Memory)     │  │  │  │   (In-Memory)     │  │  │  │   (In-Memory)     │  │
│  │   ~400 GB         │  │  │  │   ~400 GB         │  │  │  │   ~400 GB         │  │
│  └───────────────────┘  │  │  └───────────────────┘  │  │  └───────────────────┘  │
│                         │  │                         │  │                         │
│  512 GB RAM             │  │  512 GB RAM             │  │  512 GB RAM             │
│  32 cores               │  │  32 cores               │  │  32 cores               │
└─────────────────────────┘  └─────────────────────────┘  └─────────────────────────┘
              │                          │                          │
              └──────────────────────────┼──────────────────────────┘
                                         │
                                         │ Index Updates (Hourly)
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              DATA PIPELINE                                           │
│                                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │ Search Logs │───>│ Spark Jobs  │───>│ Trie Builder│───>│   S3/HDFS   │          │
│  │  (Kafka)    │    │ (Aggregate) │    │  (Hourly)   │    │ (Index File)│          │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘          │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Detailed Request Flow

### Suggestion Request Flow

```
┌──────┐     ┌─────┐     ┌─────────┐     ┌─────────────────┐
│Client│     │ CDN │     │   LB    │     │Suggestion Server│
└──┬───┘     └──┬──┘     └────┬────┘     └────────┬────────┘
   │            │             │                   │
   │ GET /suggestions?q=how   │                   │
   │───────────>│             │                   │
   │            │             │                   │
   │            │ Cache check │                   │
   │            │─────────────│                   │
   │            │             │                   │
   │            │ Cache MISS  │                   │
   │            │────────────>│                   │
   │            │             │                   │
   │            │             │ Forward to server │
   │            │             │──────────────────>│
   │            │             │                   │
   │            │             │                   │ 1. Normalize query
   │            │             │                   │────────────────────
   │            │             │                   │
   │            │             │                   │ 2. Trie lookup
   │            │             │                   │ node = trie.find("how")
   │            │             │                   │────────────────────
   │            │             │                   │
   │            │             │                   │ 3. Get pre-computed
   │            │             │                   │ suggestions
   │            │             │                   │────────────────────
   │            │             │                   │
   │            │             │                   │ 4. Apply personalization
   │            │             │                   │ (if user context)
   │            │             │                   │────────────────────
   │            │             │                   │
   │            │             │   200 OK (JSON)   │
   │            │             │<──────────────────│
   │            │             │                   │
   │            │ 200 OK      │                   │
   │            │ (cache it)  │                   │
   │            │<────────────│                   │
   │            │             │                   │
   │ 200 OK     │             │                   │
   │<───────────│             │                   │
   │            │             │                   │

Total latency: 15-50ms
- CDN: 5ms
- Network: 5-20ms  
- Trie lookup: 1-5ms
- Serialization: 1-2ms
```

---

## Trie Lookup Visualization

```
Query: "how to t"

Step 1: Navigate Trie
─────────────────────

    [root]
       │
       h ◄── Step 1
       │
       o ◄── Step 2
       │
       w ◄── Step 3
       │
     (space) ◄── Step 4
       │
       t ◄── Step 5
       │
       o ◄── Step 6
       │
     (space) ◄── Step 7
       │
       t ◄── Step 8 ★ FOUND NODE
       │
    ┌──┴──┐
    │     │
    i     r    (children: "tie", "train", etc.)
    │     │
    e     a
    │     │
   ...   ...

Step 2: Return Pre-computed Suggestions
───────────────────────────────────────

Node "how to t" has:
topSuggestions = [
    "how to tie a tie" (score: 0.95),
    "how to train your dragon" (score: 0.87),
    "how to take a screenshot" (score: 0.82),
    ...
]

Time: O(8) = O(prefix length)
```

---

## Data Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              DATA PIPELINE (Hourly)                                  │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 1: DATA COLLECTION                                                            │
│                                                                                      │
│  Search Service ───────> Kafka Topic: search-queries                                │
│                          Partitions: 100                                             │
│                          Retention: 7 days                                           │
│                                                                                      │
│  Event Format:                                                                       │
│  {                                                                                   │
│    "query": "how to tie a tie",                                                     │
│    "timestamp": 1705312800,                                                          │
│    "user_id": "abc123",                                                             │
│    "session_id": "xyz789",                                                          │
│    "result_count": 1500000                                                          │
│  }                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 2: AGGREGATION (Spark Streaming)                                              │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  // Pseudo-code                                                              │   │
│  │  searchEvents                                                                │   │
│  │    .filter(e -> !isBlocked(e.query))                                        │   │
│  │    .map(e -> normalize(e.query))                                            │   │
│  │    .groupBy("query", window("1 hour"))                                      │   │
│  │    .count()                                                                  │   │
│  │    .write("hourly_query_counts")                                            │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Output: hourly_query_counts table                                                  │
│  ┌────────────────────────────┬───────────┬─────────────┐                          │
│  │ query                      │ count     │ hour        │                          │
│  ├────────────────────────────┼───────────┼─────────────┤                          │
│  │ how to tie a tie           │ 15,234    │ 2024-01-15  │                          │
│  │ how to lose weight         │ 12,456    │ 10:00       │                          │
│  │ ...                        │ ...       │ ...         │                          │
│  └────────────────────────────┴───────────┴─────────────┘                          │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 3: SCORING & RANKING                                                          │
│                                                                                      │
│  Input: hourly_query_counts + historical_counts                                     │
│                                                                                      │
│  Score = log10(                                                                      │
│    hourly_count * 10 +                                                              │
│    daily_count * 5 +                                                                │
│    weekly_count * 2 +                                                               │
│    monthly_count * 1                                                                │
│  ) * trending_boost                                                                 │
│                                                                                      │
│  Output: Top 5 billion queries by score                                             │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 4: TRIE BUILDING                                                              │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  for each query in ranked_queries:                                           │   │
│  │      node = trie.root                                                        │   │
│  │      for each char in query:                                                 │   │
│  │          if char not in node.children:                                       │   │
│  │              node.children[char] = new TrieNode()                            │   │
│  │          node = node.children[char]                                          │   │
│  │          updateTopSuggestions(node, query, score)                            │   │
│  │      node.isEndOfQuery = true                                                │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Output: Serialized Trie (~150 GB)                                                  │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 5: DISTRIBUTION                                                               │
│                                                                                      │
│  1. Upload to S3: s3://typeahead-index/v123/trie.bin                               │
│  2. Update Zookeeper: /typeahead/current_version = "v123"                          │
│  3. Servers detect version change                                                   │
│  4. Servers download and load new Trie                                              │
│  5. Atomic switch to new Trie                                                       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Index Update Strategy

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         INDEX UPDATE (Zero Downtime)                                 │
└─────────────────────────────────────────────────────────────────────────────────────┘

Timeline:
─────────────────────────────────────────────────────────────────────────────────────

10:00  │ Pipeline starts building v124
       │
10:45  │ v124 Trie complete, uploaded to S3
       │
10:46  │ Zookeeper updated: current_version = "v124"
       │
       │ ┌─────────────────────────────────────────────────────────────────────────┐
       │ │                    SERVER ROLLING UPDATE                                │
       │ │                                                                         │
       │ │  Server 1: Serving v123 ──> Download v124 ──> Load ──> Switch ──> v124 │
       │ │  Server 2: Serving v123 ──────> Download v124 ──> Load ──> Switch ──>  │
       │ │  Server 3: Serving v123 ─────────> Download v124 ──> Load ──> Switch   │
       │ │  ...                                                                    │
       │ │                                                                         │
       │ │  Staggered: 10 servers at a time to avoid thundering herd              │
       │ └─────────────────────────────────────────────────────────────────────────┘
       │
10:55  │ All servers on v124
       │
11:00  │ Next pipeline starts building v125

Memory Management:
─────────────────────────────────────────────────────────────────────────────────────

Server RAM: 512 GB

During update:
- Old Trie (v123): 400 GB
- New Trie (v124): 400 GB (loading)
- Total: 800 GB > 512 GB ✗

Solution: Stream loading
- Load new Trie to disk first
- Memory-map the file
- Swap pointers atomically
- Old Trie garbage collected
```

---

## Caching Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              MULTI-LAYER CACHING                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘

                              ┌───────────────────┐
                              │      CLIENT       │
                              │  (Browser Cache)  │
                              │  TTL: 1 minute    │
                              └─────────┬─────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 1: CDN EDGE                                                                   │
│                                                                                      │
│  Cache Key: /suggestions?q={prefix}&limit={n}                                       │
│  TTL: 5 minutes                                                                      │
│  Hit Rate: ~40% (popular prefixes like "how", "what", "why")                        │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  Popular cached prefixes:                                                    │   │
│  │  - "how to" (millions of requests/hour)                                     │   │
│  │  - "what is" (millions of requests/hour)                                    │   │
│  │  - "weather" (varies by season)                                             │   │
│  │  - [trending topics] (dynamic)                                              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ Cache MISS (~60%)
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 2: IN-MEMORY TRIE (Per Server)                                                │
│                                                                                      │
│  - Full Trie loaded in memory                                                       │
│  - Pre-computed top suggestions at each node                                        │
│  - Lookup: O(prefix length) ≈ O(20)                                                 │
│  - No external calls needed                                                          │
│                                                                                      │
│  This IS the cache - no separate caching layer needed!                              │
└─────────────────────────────────────────────────────────────────────────────────────┘

Why no Redis/Memcached?
─────────────────────────────────────────────────────────────────────────────────────

Traditional approach:
  Client → CDN → LB → App → Redis → Response
  Latency: 5 + 5 + 1 + 5 = 16ms minimum

Our approach:
  Client → CDN → LB → App (in-memory Trie) → Response
  Latency: 5 + 5 + 1 = 11ms

Redis would add latency without benefit since:
- Trie fits in memory
- All servers have full copy
- No cold start problem
```

---

## Geographic Distribution

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              GLOBAL DEPLOYMENT                                       │
└─────────────────────────────────────────────────────────────────────────────────────┘

                         ┌─────────────────────────────┐
                         │      GLOBAL DNS (Route53)   │
                         │   Latency-based routing     │
                         └─────────────┬───────────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         │                             │                             │
         ▼                             ▼                             ▼
┌─────────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐
│    US-EAST-1        │    │    EU-WEST-1        │    │    AP-SOUTH-1       │
│                     │    │                     │    │                     │
│  ┌───────────────┐  │    │  ┌───────────────┐  │    │  ┌───────────────┐  │
│  │ CDN Edge      │  │    │  │ CDN Edge      │  │    │  │ CDN Edge      │  │
│  └───────┬───────┘  │    │  └───────┬───────┘  │    │  └───────┬───────┘  │
│          │          │    │          │          │    │          │          │
│  ┌───────┴───────┐  │    │  ┌───────┴───────┐  │    │  ┌───────┴───────┐  │
│  │ Load Balancer │  │    │  │ Load Balancer │  │    │  │ Load Balancer │  │
│  └───────┬───────┘  │    │  └───────┬───────┘  │    │  └───────┬───────┘  │
│          │          │    │          │          │    │          │          │
│  ┌───────┴───────┐  │    │  ┌───────┴───────┐  │    │  ┌───────┴───────┐  │
│  │ Suggestion    │  │    │  │ Suggestion    │  │    │  │ Suggestion    │  │
│  │ Servers (20)  │  │    │  │ Servers (20)  │  │    │  │ Servers (20)  │  │
│  │               │  │    │  │               │  │    │  │               │  │
│  │ Same Trie     │  │    │  │ Same Trie     │  │    │  │ Same Trie     │  │
│  │ (replicated)  │  │    │  │ (replicated)  │  │    │  │ (replicated)  │  │
│  └───────────────┘  │    │  └───────────────┘  │    │  └───────────────┘  │
│                     │    │                     │    │                     │
│  Serves: Americas   │    │  Serves: Europe,    │    │  Serves: Asia,      │
│                     │    │  Africa             │    │  Oceania            │
└─────────────────────┘    └─────────────────────┘    └─────────────────────┘

Index Distribution:
─────────────────────────────────────────────────────────────────────────────────────

Pipeline (US-EAST-1) ──> S3 (US-EAST-1) ──┬──> S3 (EU-WEST-1) ──> EU Servers
                                          │
                                          └──> S3 (AP-SOUTH-1) ──> AP Servers

Cross-region replication: ~5 minutes
Total update propagation: ~15 minutes globally
```

---

## Failure Scenarios

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              FAILURE ANALYSIS                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

Scenario                    Impact                      Mitigation
─────────────────────────────────────────────────────────────────────────────────────

Single server crash         Minimal (1/60 capacity)     LB routes to healthy servers
                                                        K8s restarts pod

Entire AZ failure           33% capacity loss           Multi-AZ deployment
                                                        DNS failover to other AZs

CDN edge failure            Increased origin load       Multiple edge locations
                                                        Automatic failover

Pipeline failure            Stale suggestions           Keep serving old index
                                                        Alert, manual intervention

S3 unavailable              Cannot update index         Servers keep current index
                                                        Retry with backoff

Zookeeper failure           No index updates            Servers keep current index
                                                        Manual update if needed

Memory exhaustion           Server crash                Memory limits, monitoring
                                                        Graceful degradation

Network partition           Region isolated             DNS removes unhealthy region
                                                        Users routed elsewhere
```

---

## Summary

| Component | Technology | Purpose |
|-----------|------------|---------|
| CDN | CloudFront | Edge caching for popular queries |
| Load Balancer | NLB (Layer 4) | High-throughput distribution |
| Suggestion Service | Java + Trie | In-memory prefix lookup |
| Data Pipeline | Spark | Aggregate and build index |
| Storage | S3 | Index distribution |
| Coordination | Zookeeper | Version management |
| DNS | Route53 | Latency-based routing |

