# Pastebin - Architecture Diagrams

## Component Overview

| Component | Purpose | Why It Exists |
|-----------|---------|---------------|
| **Load Balancer** | Distributes traffic | High availability, SSL termination |
| **API Gateway** | Rate limiting, auth | Centralizes cross-cutting concerns |
| **Paste Service** | Core business logic | Create, retrieve, delete pastes |
| **Metadata DB (PostgreSQL)** | Paste metadata | Fast queries, relationships |
| **Object Storage (S3)** | Paste content | Scalable, cost-effective for large files |
| **Cache (Redis)** | Hot content caching | Reduce S3 reads for popular pastes |
| **CDN** | Edge caching | Global low-latency access |
| **Cleanup Worker** | Expiration handling | Delete expired pastes |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    CLIENTS                                           │
│                         (Web Browsers, CLI Tools, APIs)                              │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                    ┌────────────────────┴────────────────────┐
                    │                                         │
                    ▼                                         ▼
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│         CDN (CloudFront)        │         │       API Gateway               │
│                                 │         │                                 │
│  - Edge caching for raw content │         │  - Rate limiting               │
│  - Static assets                │         │  - Authentication              │
│  - Geographic distribution      │         │  - Request routing             │
└────────────────┬────────────────┘         └────────────────┬────────────────┘
                 │                                           │
                 │ Cache Miss                                │
                 └──────────────────────┬────────────────────┘
                                        │
                                        ▼
                          ┌─────────────────────────────┐
                          │      LOAD BALANCER          │
                          │        (AWS ALB)            │
                          └─────────────┬───────────────┘
                                        │
              ┌─────────────────────────┼─────────────────────────┐
              ▼                         ▼                         ▼
       ┌─────────────┐           ┌─────────────┐           ┌─────────────┐
       │   Paste     │           │   Paste     │           │   Paste     │
       │  Service 1  │           │  Service 2  │           │  Service N  │
       └──────┬──────┘           └──────┬──────┘           └──────┬──────┘
              │                         │                         │
              └─────────────────────────┼─────────────────────────┘
                                        │
         ┌──────────────────────────────┼──────────────────────────────┐
         │                              │                              │
         ▼                              ▼                              ▼
┌─────────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐
│   Redis Cluster     │    │     PostgreSQL      │    │    S3 (Content)     │
│                     │    │                     │    │                     │
│ - Hot content cache │    │ - Paste metadata    │    │ - Paste content     │
│ - Rate limit data   │    │ - User data         │    │ - Compressed (gzip) │
│ - Session data      │    │ - API keys          │    │ - Deduplicated      │
└─────────────────────┘    └─────────────────────┘    └─────────────────────┘
                                        │
                                        │ Async
                                        ▼
                          ┌─────────────────────────────┐
                          │      Cleanup Worker         │
                          │                             │
                          │  - Delete expired pastes    │
                          │  - Clean orphaned content   │
                          │  - Update statistics        │
                          └─────────────────────────────┘
```

---

## Detailed Data Flow

### Create Paste Flow

```
┌──────┐     ┌─────────┐     ┌─────────────┐     ┌───────┐     ┌──────────┐     ┌─────┐
│Client│     │API Gate │     │Paste Service│     │ Redis │     │PostgreSQL│     │ S3  │
└──┬───┘     └────┬────┘     └──────┬──────┘     └───┬───┘     └────┬─────┘     └──┬──┘
   │              │                 │                │              │              │
   │ POST /pastes │                 │                │              │              │
   │─────────────>│                 │                │              │              │
   │              │                 │                │              │              │
   │              │ Rate limit check│                │              │              │
   │              │────────────────>│                │              │              │
   │              │                 │ INCR ratelimit │              │              │
   │              │                 │───────────────>│              │              │
   │              │                 │    count       │              │              │
   │              │                 │<───────────────│              │              │
   │              │                 │                │              │              │
   │              │ Forward request │                │              │              │
   │              │────────────────>│                │              │              │
   │              │                 │                │              │              │
   │              │                 │ 1. Generate ID │              │              │
   │              │                 │────────────────│              │              │
   │              │                 │                │              │              │
   │              │                 │ 2. Hash content│              │              │
   │              │                 │────────────────│              │              │
   │              │                 │                │              │              │
   │              │                 │ 3. Check dedup │              │              │
   │              │                 │──────────────────────────────>│              │
   │              │                 │                │              │              │
   │              │                 │ 4. Upload content (if new)    │              │
   │              │                 │─────────────────────────────────────────────>│
   │              │                 │                │              │     OK       │
   │              │                 │<─────────────────────────────────────────────│
   │              │                 │                │              │              │
   │              │                 │ 5. Save metadata              │              │
   │              │                 │──────────────────────────────>│              │
   │              │                 │                │     OK       │              │
   │              │                 │<──────────────────────────────│              │
   │              │                 │                │              │              │
   │              │                 │ 6. Cache metadata             │              │
   │              │                 │───────────────>│              │              │
   │              │                 │                │              │              │
   │              │  201 Created    │                │              │              │
   │              │<────────────────│                │              │              │
   │              │                 │                │              │              │
   │ 201 Created  │                 │                │              │              │
   │<─────────────│                 │                │              │              │
```

### View Paste Flow (Cache Hit)

```
┌──────┐     ┌─────┐     ┌─────────────┐     ┌───────┐
│Client│     │ CDN │     │Paste Service│     │ Redis │
└──┬───┘     └──┬──┘     └──────┬──────┘     └───┬───┘
   │            │               │                │
   │ GET /raw/abc123            │                │
   │───────────>│               │                │
   │            │               │                │
   │            │ Cache HIT     │                │
   │            │───────────────│                │
   │            │               │                │
   │ 200 OK     │               │                │
   │ (content)  │               │                │
   │<───────────│               │                │
   │            │               │                │
   
Latency: ~10ms (CDN edge)
```

### View Paste Flow (Cache Miss)

```
┌──────┐     ┌─────┐     ┌─────────────┐     ┌───────┐     ┌──────────┐     ┌─────┐
│Client│     │ CDN │     │Paste Service│     │ Redis │     │PostgreSQL│     │ S3  │
└──┬───┘     └──┬──┘     └──────┬──────┘     └───┬───┘     └────┬─────┘     └──┬──┘
   │            │               │                │              │              │
   │ GET /raw/abc123            │                │              │              │
   │───────────>│               │                │              │              │
   │            │               │                │              │              │
   │            │ Cache MISS    │                │              │              │
   │            │──────────────>│                │              │              │
   │            │               │                │              │              │
   │            │               │ 1. Check Redis │              │              │
   │            │               │───────────────>│              │              │
   │            │               │    MISS        │              │              │
   │            │               │<───────────────│              │              │
   │            │               │                │              │              │
   │            │               │ 2. Get metadata│              │              │
   │            │               │──────────────────────────────>│              │
   │            │               │    paste info  │              │              │
   │            │               │<──────────────────────────────│              │
   │            │               │                │              │              │
   │            │               │ 3. Get content │              │              │
   │            │               │─────────────────────────────────────────────>│
   │            │               │    content     │              │              │
   │            │               │<─────────────────────────────────────────────│
   │            │               │                │              │              │
   │            │               │ 4. Cache in Redis             │              │
   │            │               │───────────────>│              │              │
   │            │               │                │              │              │
   │            │  200 OK       │                │              │              │
   │            │  (cache it)   │                │              │              │
   │            │<──────────────│                │              │              │
   │            │               │                │              │              │
   │ 200 OK     │               │                │              │              │
   │<───────────│               │                │              │              │

Latency: ~100ms (origin + S3)
```

---

## Caching Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              MULTI-LAYER CACHING                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘

                              ┌───────────────────┐
                              │      CLIENT       │
                              └─────────┬─────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 1: CDN (CloudFront)                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  Cache Key: /raw/{paste_id}                                                  │   │
│  │  TTL: 1 hour for public, 0 for private                                       │   │
│  │  Hit Rate: ~60% for popular pastes                                           │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ Cache MISS
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 2: Application Cache (Redis)                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                              │   │
│  │  Metadata Cache:                     Content Cache:                          │   │
│  │  ┌─────────────────────────┐        ┌─────────────────────────┐             │   │
│  │  │ Key: meta:{paste_id}    │        │ Key: content:{paste_id} │             │   │
│  │  │ Value: JSON metadata    │        │ Value: compressed text  │             │   │
│  │  │ TTL: 1 hour             │        │ TTL: 15 minutes         │             │   │
│  │  │ Size: ~500 bytes        │        │ Max Size: 1 MB          │             │   │
│  │  └─────────────────────────┘        └─────────────────────────┘             │   │
│  │                                                                              │   │
│  │  Only cache pastes < 1 MB in Redis (larger go directly to S3)               │   │
│  │  Hit Rate: ~40%                                                              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ Cache MISS
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 3: Origin Storage                                                            │
│  ┌───────────────────────────────────┐  ┌───────────────────────────────────┐      │
│  │         PostgreSQL                │  │            S3                     │      │
│  │                                   │  │                                   │      │
│  │  - Paste metadata                 │  │  - Paste content                  │      │
│  │  - Always authoritative           │  │  - Compressed (gzip)              │      │
│  │  - Query for existence check      │  │  - Lifecycle policies             │      │
│  └───────────────────────────────────┘  └───────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Cache Decision Logic

```java
public class CacheStrategy {
    
    private static final long MAX_REDIS_SIZE = 1024 * 1024; // 1 MB
    
    public CacheDecision decide(Paste paste) {
        // Private pastes: no CDN caching
        if (paste.getVisibility() == Visibility.PRIVATE) {
            return CacheDecision.builder()
                .cdnCacheable(false)
                .redisCacheable(true)
                .redisTtl(Duration.ofMinutes(5))
                .build();
        }
        
        // Large pastes: skip Redis, only CDN
        if (paste.getSizeBytes() > MAX_REDIS_SIZE) {
            return CacheDecision.builder()
                .cdnCacheable(true)
                .cdnTtl(Duration.ofHours(1))
                .redisCacheable(false)
                .build();
        }
        
        // Normal pastes: cache everywhere
        return CacheDecision.builder()
            .cdnCacheable(true)
            .cdnTtl(Duration.ofHours(1))
            .redisCacheable(true)
            .redisTtl(Duration.ofMinutes(15))
            .build();
    }
}
```

---

## Storage Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              STORAGE ARCHITECTURE                                    │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              PostgreSQL (Metadata)                                   │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                              pastes table                                    │   │
│  │  ┌──────────┬──────────────────┬───────────┬──────────┬─────────────────┐   │   │
│  │  │    id    │   storage_key    │   title   │  syntax  │   expires_at    │   │   │
│  │  ├──────────┼──────────────────┼───────────┼──────────┼─────────────────┤   │   │
│  │  │ abc12345 │ pastes/2024/...  │ My Code   │ python   │ 2024-02-15      │   │   │
│  │  │ def67890 │ dedup/sha256-... │ Config    │ yaml     │ NULL (never)    │   │   │
│  │  └──────────┴──────────────────┴───────────┴──────────┴─────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Primary: us-east-1a                                                                 │
│  Replicas: us-east-1b, us-west-2a                                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ storage_key references
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              S3 (Content Storage)                                    │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  Bucket: pastebin-content-prod                                               │   │
│  │                                                                              │   │
│  │  pastes/                                                                     │   │
│  │  ├── 2024/                                                                   │   │
│  │  │   ├── 01/                                                                 │   │
│  │  │   │   ├── 15/                                                             │   │
│  │  │   │   │   ├── abc12345.txt.gz  (45 bytes compressed)                      │   │
│  │  │   │   │   └── ghi34567.txt.gz  (1.2 KB compressed)                        │   │
│  │  │   │   └── 16/                                                             │   │
│  │  │   │       └── ...                                                         │   │
│  │  │   └── 02/                                                                 │   │
│  │  │       └── ...                                                             │   │
│  │  │                                                                           │   │
│  │  deduplicated/                                                               │   │
│  │  └── sha256-a1b2c3d4...xyz.txt.gz  (shared by multiple pastes)              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Storage Class: S3 Standard (first 30 days) → S3 IA (after 30 days)                 │
│  Encryption: SSE-S3                                                                  │
│  Versioning: Disabled (pastes are immutable)                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### S3 Lifecycle Policy

```json
{
  "Rules": [
    {
      "ID": "TransitionToIA",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "pastes/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        }
      ]
    },
    {
      "ID": "DeleteExpired",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "pastes/"
      },
      "Expiration": {
        "Days": 365
      }
    }
  ]
}
```

---

## Cleanup Worker Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              CLEANUP WORKER                                          │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                      │
│  ┌─────────────────┐                                                                │
│  │   Scheduler     │  Runs every hour                                               │
│  │   (Cron)        │                                                                │
│  └────────┬────────┘                                                                │
│           │                                                                          │
│           ▼                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         Cleanup Job                                          │   │
│  │                                                                              │   │
│  │  1. Query expired pastes (batch of 1000)                                     │   │
│  │     SELECT id, storage_key FROM pastes                                       │   │
│  │     WHERE expires_at < NOW() AND deleted_at IS NULL                          │   │
│  │     LIMIT 1000                                                               │   │
│  │                                                                              │   │
│  │  2. For each paste:                                                          │   │
│  │     a. Soft delete in PostgreSQL (set deleted_at)                            │   │
│  │     b. Delete from Redis cache                                               │   │
│  │     c. Invalidate CDN cache                                                  │   │
│  │     d. Queue S3 deletion (async)                                             │   │
│  │                                                                              │   │
│  │  3. Process S3 deletions in batches                                          │   │
│  │     (S3 DeleteObjects API supports 1000 keys per request)                    │   │
│  │                                                                              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│           │                                                                          │
│           ▼                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                     Orphan Content Cleanup (Weekly)                          │   │
│  │                                                                              │   │
│  │  Find S3 objects with no corresponding paste record                         │   │
│  │  (handles crashes during paste creation)                                     │   │
│  │                                                                              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Deployment Architecture (Kubernetes)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              KUBERNETES CLUSTER                                      │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                              INGRESS (NGINX)                                   │  │
│  │                         TLS termination, routing                               │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                        │                                             │
│         ┌──────────────────────────────┼──────────────────────────────┐             │
│         ▼                              ▼                              ▼             │
│  ┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐     │
│  │ paste-service   │          │ cleanup-worker  │          │ web-frontend    │     │
│  │ Deployment      │          │ CronJob         │          │ Deployment      │     │
│  │                 │          │                 │          │                 │     │
│  │ Replicas: 5     │          │ Schedule: hourly│          │ Replicas: 3     │     │
│  │ CPU: 1 core     │          │ CPU: 0.5 core   │          │ CPU: 0.5 core   │     │
│  │ Memory: 2Gi     │          │ Memory: 1Gi     │          │ Memory: 512Mi   │     │
│  └─────────────────┘          └─────────────────┘          └─────────────────┘     │
│           │                                                                          │
│  ┌────────┴────────────────────────────────────────────────────────────────────┐    │
│  │                           SERVICES (ClusterIP)                               │    │
│  │                                                                              │    │
│  │  paste-service:8080              web-frontend:3000                           │    │
│  └──────────────────────────────────────────────────────────────────────────────┘    │
│                                        │                                             │
│  ┌─────────────────────────────────────┴──────────────────────────────────────┐     │
│  │                              EXTERNAL SERVICES                              │     │
│  │                                                                             │     │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐     │     │
│  │  │ RDS PostgreSQL  │  │ ElastiCache     │  │ S3 Bucket               │     │     │
│  │  │                 │  │ (Redis)         │  │                         │     │     │
│  │  │ Multi-AZ        │  │ Cluster Mode    │  │ pastebin-content-prod   │     │     │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────────┘     │     │
│  └─────────────────────────────────────────────────────────────────────────────┘     │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Failure Points and Recovery

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              FAILURE ANALYSIS                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

Component              Failure Mode           Impact              Recovery
─────────────────────────────────────────────────────────────────────────────────────

┌─────────────┐
│     CDN     │ ───── Edge down ──────── Minor latency ──── Auto-failover to
│             │                          increase           other edges
└─────────────┘

┌─────────────┐
│   Paste     │ ───── Pod crash ──────── Degraded ───────── K8s auto-restart
│   Service   │                          capacity           + HPA scaling
└─────────────┘

┌─────────────┐
│    Redis    │ ───── Node down ──────── Cache miss ─────── Cluster failover
│             │                          spike, S3 load     (automatic)
└─────────────┘

┌─────────────┐
│ PostgreSQL  │ ───── Primary down ───── Writes fail ────── Promote replica
│             │                          (30 seconds)       (automatic)
└─────────────┘

┌─────────────┐
│     S3      │ ───── Region down ────── Content ────────── S3 cross-region
│             │                          unavailable        replication
└─────────────┘

┌─────────────┐
│   Cleanup   │ ───── Job fails ──────── Expired pastes ─── Retry on next
│   Worker    │                          accumulate         schedule
└─────────────┘
```

### Graceful Degradation

```java
public class PasteService {
    
    public PasteResponse getPaste(String id) {
        try {
            // Try full path
            return getFromCacheOrStorage(id);
        } catch (StorageException e) {
            // S3 is down, try Redis only
            Paste cached = redis.get("content:" + id);
            if (cached != null) {
                return PasteResponse.fromCache(cached);
            }
            throw new ServiceUnavailableException("Content temporarily unavailable");
        } catch (DatabaseException e) {
            // PostgreSQL is down, serve from cache if available
            Paste cached = redis.get("meta:" + id);
            if (cached != null) {
                return PasteResponse.fromCache(cached);
            }
            throw new ServiceUnavailableException("Service temporarily unavailable");
        }
    }
}
```

---

## Summary

| Component | Technology | Purpose |
|-----------|------------|---------|
| CDN | CloudFront | Edge caching, global distribution |
| Load Balancer | AWS ALB | Traffic distribution, health checks |
| API Service | Spring Boot | Business logic |
| Metadata DB | PostgreSQL | Paste metadata, user data |
| Content Storage | S3 | Paste content (compressed) |
| Cache | Redis Cluster | Hot content, rate limiting |
| Cleanup | K8s CronJob | Expire old pastes |
| Monitoring | Prometheus + Grafana | Metrics, alerting |

