# File Storage System - Production Deep Dives (Core)

## 1. Kafka Deep Dive: Event-Driven Sync

### Why Kafka for File Storage?

```
Problem: How do we notify all devices when a file changes?

Without Kafka:
├── Polling: Each device polls every 30 seconds
├── 5M concurrent users × 2 req/min = 167K req/sec
├── Wasteful: 99% of polls return "no changes"
└── Latency: Up to 30 seconds delay

With Kafka:
├── Publish: File change → Kafka event
├── Subscribe: Devices receive push notification
├── Efficient: Only actual changes transmitted
└── Latency: < 5 seconds
```

### Topic Design

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Kafka Topic Design                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Topic: file-events                                                           │
│  ├── Partitions: 100                                                         │
│  ├── Replication: 3                                                          │
│  ├── Retention: 7 days                                                       │
│  └── Key: user_id (ensures ordering per user)                                │
│                                                                               │
│  Event Types:                                                                 │
│  ├── FILE_CREATED                                                            │
│  ├── FILE_MODIFIED                                                           │
│  ├── FILE_DELETED                                                            │
│  ├── FILE_MOVED                                                              │
│  ├── FILE_SHARED                                                             │
│  └── FOLDER_CHANGED                                                          │
│                                                                               │
│  Topic: sync-notifications                                                    │
│  ├── Partitions: 50                                                          │
│  ├── Key: device_id                                                          │
│  └── Purpose: Push to specific devices                                       │
│                                                                               │
│  Topic: search-index                                                          │
│  ├── Partitions: 20                                                          │
│  └── Purpose: Feed Elasticsearch indexer                                     │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Event Schema

```java
public class FileEvent {
    private String eventId;          // UUID for idempotency
    private FileEventType type;      // CREATED, MODIFIED, DELETED, etc.
    private String userId;           // File owner
    private String fileId;
    private String path;
    private String newPath;          // For moves/renames
    private long size;
    private String contentHash;
    private String rev;              // Revision for sync
    private Instant timestamp;
    private String sourceDeviceId;   // Which device made the change
}

// Producer
@Service
public class FileEventProducer {
    
    private final KafkaTemplate<String, FileEvent> kafka;
    
    public void publishFileCreated(File file, String sourceDevice) {
        FileEvent event = FileEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .type(FileEventType.CREATED)
            .userId(file.getOwnerId())
            .fileId(file.getId())
            .path(file.getPath())
            .size(file.getSize())
            .contentHash(file.getContentHash())
            .rev(file.getRev())
            .timestamp(Instant.now())
            .sourceDeviceId(sourceDevice)
            .build();
        
        // Key by userId for ordering
        kafka.send("file-events", file.getOwnerId(), event);
    }
}
```

### Consumer Groups

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Consumer Group Design                                 │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Consumer Group: sync-notifier                                                │
│  ├── Purpose: Notify connected devices of changes                            │
│  ├── Consumers: 20 instances                                                 │
│  ├── Processing: Push via WebSocket                                          │
│  └── Latency requirement: < 1 second                                         │
│                                                                               │
│  Consumer Group: search-indexer                                               │
│  ├── Purpose: Update Elasticsearch index                                     │
│  ├── Consumers: 10 instances                                                 │
│  ├── Processing: Batch index updates                                         │
│  └── Latency requirement: < 5 minutes                                        │
│                                                                               │
│  Consumer Group: activity-logger                                              │
│  ├── Purpose: Record activity for audit                                      │
│  ├── Consumers: 5 instances                                                  │
│  ├── Processing: Write to activity DB                                        │
│  └── Latency requirement: < 1 minute                                         │
│                                                                               │
│  Consumer Group: quota-updater                                                │
│  ├── Purpose: Update storage quota                                           │
│  ├── Consumers: 5 instances                                                  │
│  └── Processing: Increment/decrement user quota                              │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Sync Notification Flow

```java
@Service
public class SyncNotificationConsumer {
    
    private final WebSocketSessionManager sessionManager;
    private final DeviceService deviceService;
    
    @KafkaListener(
        topics = "file-events",
        groupId = "sync-notifier",
        concurrency = "20"
    )
    public void onFileEvent(FileEvent event) {
        // Get all devices for this user except source
        List<Device> devices = deviceService.getActiveDevices(event.getUserId())
            .stream()
            .filter(d -> !d.getId().equals(event.getSourceDeviceId()))
            .collect(Collectors.toList());
        
        // Send notification to each connected device
        for (Device device : devices) {
            WebSocketSession session = sessionManager.getSession(device.getId());
            if (session != null && session.isOpen()) {
                // Push notification
                SyncNotification notification = SyncNotification.builder()
                    .type("sync_required")
                    .path(event.getPath())
                    .build();
                session.sendMessage(new TextMessage(toJson(notification)));
            } else {
                // Device not connected, will sync on next poll
                log.debug("Device {} not connected", device.getId());
            }
        }
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Kafka Technology-Level

**Normal Flow: File Change Event Propagation**

```
Step 1: File Modified
┌─────────────────────────────────────────────────────────────┐
│ User edits file "report.docx" on laptop                      │
│ File Service: Detects change, updates metadata               │
│ Kafka Producer: Publishes FILE_MODIFIED event                 │
│   Topic: file-events                                         │
│   Partition: hash(user_123) % 100 = partition 45              │
│   Message: {                                                  │
│     "eventId": "evt_001",                                     │
│     "type": "FILE_MODIFIED",                                 │
│     "userId": "user_123",                                    │
│     "fileId": "file_456",                                     │
│     "path": "/Documents/report.docx",                        │
│     "rev": "v7",                                             │
│     "timestamp": "2024-01-20T10:00:00Z"                      │
│   }                                                           │
│ Latency: < 1ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Sync Notification Consumer
┌─────────────────────────────────────────────────────────────┐
│ Consumer Group: sync-notifiers                               │
│ Worker assigned to partition 45 polls:                      │
│ - Receives batch of 10 events                                │
│ - Filters: Get events for user_123's devices                │
│ - Finds devices: [laptop_1, phone_1, tablet_1]             │
│ - Publishes to sync-notifications topic (per device)        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Device Notification Workers
┌─────────────────────────────────────────────────────────────┐
│ For each device:                                             │
│ - Consumer: device-notifier-{device_id}                     │
│ - Receives notification event                               │
│ - Checks if device is online (WebSocket connection)        │
│ - If online: Push via WebSocket                             │
│ - If offline: Queue for next connection                     │
│ Total latency: < 5 seconds                                   │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Kafka Consumer Lag**

```
Scenario: Consumer falls behind during traffic spike

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Normal operation (100 events/second)                  │
│ T+60s: Traffic spike (10,000 events/second)                 │
│ T+61s: Consumer can only process 1,000 events/second       │
│ T+62s: Lag builds: 9,000 events/second                      │
│ T+300s: Lag reaches 2.7M events                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Mitigation:                                                  │
│ 1. Auto-scale consumers: 1 → 10 instances                  │
│ 2. Increase batch size: 100 → 1,000 events                  │
│ 3. Parallel processing: Process multiple partitions         │
│ 4. Result: Throughput increases to 10,000 events/second    │
│ 5. Lag clears within 5 minutes                              │
└─────────────────────────────────────────────────────────────┘
```

**Idempotency Handling:**

```
Problem: Same file event processed twice (consumer retry)

Solution: Event ID deduplication
1. Each event has unique eventId (UUID)
2. Consumer tracks processed eventIds (Redis set)
3. Duplicate events (same eventId) ignored
4. File service operations are idempotent

Example:
- Event processed twice → Same eventId → Deduplicated → Idempotent
```

**Traffic Spike Handling:**

```
Scenario: Millions of files synced after outage

┌─────────────────────────────────────────────────────────────┐
│ Normal: 1,000 file events/second                             │
│ Spike: 100,000 file events/second (reconnection storm)    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Kafka Buffering:                                             │
│ - 100K events/second buffered in Kafka                      │
│ - 7-day retention = 60B events capacity                     │
│ - No message loss                                            │
│                                                              │
│ Consumer Scaling:                                            │
│ - Auto-scale: 10 → 100 consumer instances                   │
│ - Each consumer: 1,000 events/second                        │
│ - Throughput: 100K events/second                            │
│                                                              │
│ Result:                                                      │
│ - Events queued in Kafka (no loss)                          │
│ - Processing delayed but continues                           │
│ - All files eventually synced                                │
└─────────────────────────────────────────────────────────────┘
```

**Hot Partition Mitigation:**

```
Problem: Popular user makes many changes, all events go to one partition

Mitigation:
1. Partition by userId (distributes across users)
2. Multiple consumers per partition (if needed)
3. Monitor partition lag per partition

If hot partition occurs:
- Partition receives 1,000 events/second (active user)
- Consumers auto-scale to handle load
- Consider priority queue for high-volume users
```

**Recovery Behavior:**

```
Auto-healing:
- Consumer crash → Kafka rebalance → Events reassigned → Retry
- Network timeout → Exponential backoff → Retry
- Processing failure → Dead letter queue → Alert ops

Human intervention:
- Persistent lag → Scale consumers or optimize processing
- Dead letter queue growth → Investigate and fix processing logic
- Partition imbalance → Rebalance partitions
```

### Failure Scenarios

```
Scenario 1: Consumer Falls Behind
┌─────────────────────────────────────────────────────────────────────────┐
│  Symptoms:                                                               │
│  • Consumer lag increasing                                               │
│  • Sync notifications delayed                                            │
│                                                                          │
│  Detection:                                                              │
│  • Monitor consumer lag metric                                           │
│  • Alert when lag > 10,000 messages                                     │
│                                                                          │
│  Resolution:                                                             │
│  1. Scale up consumers (more instances)                                 │
│  2. If lag > 1 hour, consider skipping old events                       │
│  3. Clients will catch up via delta sync                                │
└─────────────────────────────────────────────────────────────────────────┘

Scenario 2: Duplicate Events
┌─────────────────────────────────────────────────────────────────────────┐
│  Cause: Consumer crash after processing, before commit                   │
│                                                                          │
│  Prevention:                                                             │
│  • Idempotent event IDs                                                 │
│  • Deduplication in consumers                                           │
│                                                                          │
│  Implementation:                                                         │
│  ```java                                                                │
│  @KafkaListener(...)                                                    │
│  public void onEvent(FileEvent event) {                                 │
│      // Check if already processed                                      │
│      if (processedEvents.contains(event.getEventId())) {               │
│          return; // Skip duplicate                                      │
│      }                                                                   │
│      // Process event                                                   │
│      processEvent(event);                                               │
│      // Mark as processed (with TTL)                                    │
│      processedEvents.add(event.getEventId(), Duration.ofHours(24));    │
│  }                                                                      │
│  ```                                                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Redis Deep Dive: Metadata Caching

### Why Redis for File Storage?

```
Problem: Database can't handle metadata read load

Without Redis:
├── 200K metadata requests/sec at peak
├── Each request: 5-10ms database query
├── Database CPU: 100% (overloaded)
└── Latency: 50-100ms

With Redis:
├── Cache hit rate: 85%
├── Redis latency: < 1ms
├── Database load: 30K requests/sec
└── Latency: < 5ms (average)
```

### Cache Strategy

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Redis Cache Strategy                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Pattern: Cache-Aside with Write-Through Invalidation                        │
│                                                                               │
│  Read Flow:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  1. Check Redis cache                                                │     │
│  │  2. If hit → Return cached data                                     │     │
│  │  3. If miss → Query PostgreSQL                                      │     │
│  │  4. Store in Redis with TTL                                         │     │
│  │  5. Return data                                                      │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  Write Flow:                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  1. Update PostgreSQL (source of truth)                             │     │
│  │  2. Delete from Redis (invalidate)                                  │     │
│  │  3. Publish invalidation event (for other instances)               │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Cache Key Design

```java
public class CacheKeys {
    
    // File metadata by ID
    // TTL: 15 minutes
    public static String fileById(String fileId) {
        return "file:id:" + fileId;
    }
    
    // File metadata by path (case-insensitive)
    // TTL: 15 minutes
    public static String fileByPath(String userId, String path) {
        return "file:path:" + userId + ":" + path.toLowerCase();
    }
    
    // Folder contents listing
    // TTL: 5 minutes (changes more frequently)
    public static String folderContents(String folderId, String sortBy) {
        return "folder:contents:" + folderId + ":" + sortBy;
    }
    
    // User's root folder ID
    // TTL: 1 hour (rarely changes)
    public static String userRoot(String userId) {
        return "user:root:" + userId;
    }
    
    // User storage quota
    // TTL: 1 minute (updated frequently)
    public static String userQuota(String userId) {
        return "user:quota:" + userId;
    }
    
    // Sync cursor
    // TTL: 24 hours
    public static String syncCursor(String userId, String deviceId) {
        return "sync:cursor:" + userId + ":" + deviceId;
    }
    
    // Share link data
    // TTL: 1 hour
    public static String shareLink(String shortCode) {
        return "share:link:" + shortCode;
    }
}
```

### Implementation

```java
@Service
public class FileMetadataCache {
    
    private final RedisTemplate<String, Object> redis;
    private final FileRepository fileRepository;
    
    private static final Duration FILE_TTL = Duration.ofMinutes(15);
    private static final Duration FOLDER_TTL = Duration.ofMinutes(5);
    
    private final DistributedLock lockService;
    private static final double EARLY_EXPIRATION_PROBABILITY = 0.01; // 1%
    
    public File getFileById(String fileId) {
        String key = CacheKeys.fileById(fileId);
        
        // Try cache first
        File cached = (File) redis.opsForValue().get(key);
        if (cached != null) {
            // Probabilistic early expiration (1% chance)
            if (Math.random() < EARLY_EXPIRATION_PROBABILITY) {
                refreshCacheAsync(fileId);
            }
            return cached;
        }
        
        // Cache miss: Try to acquire lock
        String lockKey = "lock:file:" + fileId;
        boolean acquired = lockService.tryLock(lockKey, Duration.ofSeconds(5));
        
        if (!acquired) {
            // Another thread is fetching, wait briefly
            try {
                Thread.sleep(50);
                cached = (File) redis.opsForValue().get(key);
                if (cached != null) {
                    return cached;
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            // Fall back to database (acceptable degradation)
        }
        
        try {
            // Double-check cache
            cached = (File) redis.opsForValue().get(key);
            if (cached != null) {
                return cached;
            }
            
            // Fetch from database (only one thread does this)
            File file = fileRepository.findById(fileId)
                .orElseThrow(() -> new FileNotFoundException(fileId));
            
            // Store in cache
            redis.opsForValue().set(key, file, FILE_TTL);
            
            // Also cache by path for path lookups
            redis.opsForValue().set(
                CacheKeys.fileByPath(file.getOwnerId(), file.getPath()),
                file,
                FILE_TTL
            );
            
            return file;
            
        } finally {
            if (acquired) {
                lockService.unlock(lockKey);
            }
        }
    }
    
    private void refreshCacheAsync(String fileId) {
        // Background refresh without blocking
        CompletableFuture.runAsync(() -> {
            try {
                File file = fileRepository.findById(fileId).orElse(null);
                if (file != null) {
                    String key = CacheKeys.fileById(fileId);
                    redis.opsForValue().set(key, file, FILE_TTL);
                }
            } catch (Exception e) {
                log.warn("Failed to refresh cache for file {}", fileId, e);
            }
        });
    }
    
    public File getFileByPath(String userId, String path) {
        String key = CacheKeys.fileByPath(userId, path);
        
        File cached = (File) redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // Cache miss
        File file = fileRepository.findByOwnerAndPath(userId, path.toLowerCase())
            .orElseThrow(() -> new FileNotFoundException(path));
        
        // Cache both keys
        redis.opsForValue().set(key, file, FILE_TTL);
        redis.opsForValue().set(CacheKeys.fileById(file.getId()), file, FILE_TTL);
        
        return file;
    }
    
    public void invalidateFile(File file) {
        // Delete all cache entries for this file
        redis.delete(Arrays.asList(
            CacheKeys.fileById(file.getId()),
            CacheKeys.fileByPath(file.getOwnerId(), file.getPath()),
            CacheKeys.folderContents(file.getParentId(), "name"),
            CacheKeys.folderContents(file.getParentId(), "modified_at"),
            CacheKeys.folderContents(file.getParentId(), "size")
        ));
    }
}
```

### Folder Listing Cache

```java
@Service
public class FolderListingCache {
    
    private final RedisTemplate<String, Object> redis;
    private final FileRepository fileRepository;
    
    public FolderListing getFolderContents(
            String folderId, 
            String sortBy, 
            int limit, 
            String cursor) {
        
        // For paginated results, only cache first page
        if (cursor != null) {
            return queryDatabase(folderId, sortBy, limit, cursor);
        }
        
        String key = CacheKeys.folderContents(folderId, sortBy);
        
        FolderListing cached = (FolderListing) redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // Query database
        FolderListing listing = queryDatabase(folderId, sortBy, limit, null);
        
        // Cache only if folder has reasonable number of items
        if (listing.getTotalCount() <= 1000) {
            redis.opsForValue().set(key, listing, Duration.ofMinutes(5));
        }
        
        return listing;
    }
}
```

### Cache Stampede Prevention

**What is Cache Stampede?**

Cache stampede occurs when a cached file metadata expires and many requests simultaneously try to fetch it from the database, overwhelming the database.

**Scenario:**
```
T+0s:   Popular shared file cache expires
T+0s:   10,000 requests arrive simultaneously
T+0s:   All 10,000 requests see cache MISS
T+0s:   All 10,000 requests hit database simultaneously
T+1s:   Database overwhelmed, latency spikes
```

**Prevention Strategy: Distributed Lock + Probabilistic Early Expiration**

The implementation above in `getFileById()` already includes:
1. **Probabilistic early expiration**: 1% of cache hits trigger background refresh
2. **Distributed locking**: Only one thread fetches from database on cache miss
3. **Double-check pattern**: Re-check cache after acquiring lock
4. **Background refresh**: Non-blocking cache warming

**Cache Stampede Simulation:**

```
Scenario: Popular shared file cache expires, 10,000 requests arrive

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Cache entry expires for "file_abc123"                  │
│ T+0s: 10,000 requests arrive simultaneously                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Without Protection:                                          │
│ - All 10,000 requests see cache MISS                        │
│ - All 10,000 requests hit database                          │
│ - Database overwhelmed (10,000 concurrent queries)         │
│ - Latency: 5ms → 500ms                                      │
│ - Some requests timeout                                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ With Protection (Locking):                                  │
│ - Request 1: Acquires lock, fetches from DB                │
│ - Requests 2-10,000: Lock unavailable, wait 50ms            │
│ - Request 1: Populates cache, releases lock                 │
│ - Requests 2-10,000: Retry cache → HIT                      │
│ - Database: Only 1 query (not 10,000)                      │
│ - Latency: 5ms (cache hit) for 99.99% of requests          │
└─────────────────────────────────────────────────────────────┘
```

**Probabilistic Early Expiration Benefits:**

```
Normal TTL: 15 minutes
Early expiration: 1% chance per request

Benefits:
- Cache refreshed before expiration
- No stampede (only 1% of requests trigger refresh)
- Background refresh doesn't block requests
- Cache stays warm
```

### Redis Cluster Configuration

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                       Redis Cluster Configuration                             │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Cluster Size: 200 nodes                                                      │
│  Memory per node: 64 GB                                                       │
│  Total memory: 12.8 TB                                                        │
│                                                                               │
│  Replication: 1 replica per master                                            │
│  Masters: 100                                                                 │
│  Replicas: 100                                                                │
│                                                                               │
│  Hash Slots: 16384 (distributed across masters)                               │
│                                                                               │
│  Data Distribution:                                                           │
│  ├── File metadata: 8 TB                                                     │
│  ├── Folder listings: 2 TB                                                   │
│  ├── Sync cursors: 1 TB                                                      │
│  ├── Share links: 500 GB                                                     │
│  ├── Rate limiting: 100 GB                                                   │
│  └── Sessions: 500 GB                                                        │
│                                                                               │
│  Eviction Policy: volatile-lru                                                │
│  (Evict least recently used keys with TTL set)                               │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Elasticsearch Deep Dive: File Search

### Why Elasticsearch for File Search?

```
Problem: Users need to find files by content and metadata

PostgreSQL limitations:
├── Full-text search is slow on 100B files
├── No relevance scoring
├── No fuzzy matching
├── No autocomplete
└── Can't combine text + filters efficiently

Elasticsearch provides:
├── Inverted index for fast text search
├── BM25 relevance scoring
├── Fuzzy matching for typos
├── Autocomplete with edge n-grams
├── Efficient filter + text combinations
└── Horizontal scaling
```

### Index Design

```json
{
  "settings": {
    "number_of_shards": 100,
    "number_of_replicas": 2,
    "analysis": {
      "analyzer": {
        "filename_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding", "filename_ngram"]
        },
        "content_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding", "english_stemmer"]
        }
      },
      "filter": {
        "filename_ngram": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 20
        },
        "english_stemmer": {
          "type": "stemmer",
          "language": "english"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "file_id": { "type": "keyword" },
      "owner_id": { "type": "keyword" },
      "name": {
        "type": "text",
        "analyzer": "filename_analyzer",
        "fields": {
          "keyword": { "type": "keyword" },
          "suggest": {
            "type": "completion",
            "analyzer": "simple"
          }
        }
      },
      "path": { "type": "keyword" },
      "content": {
        "type": "text",
        "analyzer": "content_analyzer"
      },
      "mime_type": { "type": "keyword" },
      "extension": { "type": "keyword" },
      "size": { "type": "long" },
      "created_at": { "type": "date" },
      "modified_at": { "type": "date" },
      "shared": { "type": "boolean" },
      "shared_with": { "type": "keyword" },
      "tags": { "type": "keyword" }
    }
  }
}
```

### Search Implementation

```java
@Service
public class FileSearchService {
    
    private final ElasticsearchClient esClient;
    
    public SearchResults search(SearchRequest request) {
        // Build query
        BoolQuery.Builder boolQuery = new BoolQuery.Builder();
        
        // Must match owner or shared with user
        boolQuery.filter(q -> q.bool(b -> b
            .should(s -> s.term(t -> t.field("owner_id").value(request.getUserId())))
            .should(s -> s.term(t -> t.field("shared_with").value(request.getUserId())))
            .minimumShouldMatch("1")
        ));
        
        // Text search across name and content
        if (request.getQuery() != null && !request.getQuery().isEmpty()) {
            boolQuery.must(q -> q.multiMatch(m -> m
                .query(request.getQuery())
                .fields("name^3", "name.keyword^5", "content", "path")
                .type(TextQueryType.BestFields)
                .fuzziness("AUTO")
            ));
        }
        
        // Apply filters
        if (request.getFileTypes() != null) {
            boolQuery.filter(q -> q.terms(t -> t
                .field("extension")
                .terms(v -> v.value(request.getFileTypes().stream()
                    .map(FieldValue::of)
                    .collect(Collectors.toList())))
            ));
        }
        
        if (request.getModifiedAfter() != null) {
            boolQuery.filter(q -> q.range(r -> r
                .field("modified_at")
                .gte(JsonData.of(request.getModifiedAfter()))
            ));
        }
        
        // Execute search
        co.elastic.clients.elasticsearch.core.SearchRequest esRequest = 
            co.elastic.clients.elasticsearch.core.SearchRequest.of(s -> s
                .index("files")
                .query(q -> q.bool(boolQuery.build()))
                .highlight(h -> h
                    .fields("content", f -> f
                        .preTags("<em>")
                        .postTags("</em>")
                        .fragmentSize(150)
                        .numberOfFragments(3)
                    )
                )
                .from(request.getOffset())
                .size(request.getLimit())
                .sort(sort -> sort.score(sc -> sc.order(SortOrder.Desc)))
        );
        
        SearchResponse<FileDocument> response = esClient.search(esRequest, FileDocument.class);
        
        return mapToSearchResults(response);
    }
    
    public List<String> autocomplete(String userId, String prefix) {
        co.elastic.clients.elasticsearch.core.SearchRequest request = 
            co.elastic.clients.elasticsearch.core.SearchRequest.of(s -> s
                .index("files")
                .query(q -> q.bool(b -> b
                    .filter(f -> f.term(t -> t.field("owner_id").value(userId)))
                    .must(m -> m.prefix(p -> p.field("name.keyword").value(prefix)))
                ))
                .size(10)
                .source(src -> src.includes(List.of("name", "path")))
        );
        
        SearchResponse<FileDocument> response = esClient.search(request, FileDocument.class);
        
        return response.hits().hits().stream()
            .map(h -> h.source().getName())
            .distinct()
            .collect(Collectors.toList());
    }
}
```

### Content Extraction Pipeline

```java
@Service
public class ContentIndexer {
    
    private final TikaParser tikaParser;
    private final ElasticsearchClient esClient;
    
    @KafkaListener(topics = "file-events", groupId = "search-indexer")
    public void onFileEvent(FileEvent event) {
        switch (event.getType()) {
            case CREATED:
            case MODIFIED:
                indexFile(event);
                break;
            case DELETED:
                deleteFromIndex(event.getFileId());
                break;
            case MOVED:
                updatePath(event.getFileId(), event.getNewPath());
                break;
        }
    }
    
    private void indexFile(FileEvent event) {
        // Download file for content extraction
        File file = fileService.getFile(event.getFileId());
        
        String content = "";
        if (isTextExtractable(file.getMimeType())) {
            try {
                InputStream stream = storageService.download(file.getStorageKey());
                content = tikaParser.extractText(stream, file.getMimeType());
                // Limit content size
                if (content.length() > 100_000) {
                    content = content.substring(0, 100_000);
                }
            } catch (Exception e) {
                log.warn("Failed to extract content from {}", file.getId(), e);
            }
        }
        
        FileDocument doc = FileDocument.builder()
            .fileId(file.getId())
            .ownerId(file.getOwnerId())
            .name(file.getName())
            .path(file.getPath())
            .content(content)
            .mimeType(file.getMimeType())
            .extension(getExtension(file.getName()))
            .size(file.getSize())
            .createdAt(file.getCreatedAt())
            .modifiedAt(file.getModifiedAt())
            .shared(!file.getShares().isEmpty())
            .sharedWith(file.getShares().stream()
                .map(Share::getUserId)
                .collect(Collectors.toList()))
            .build();
        
        esClient.index(i -> i
            .index("files")
            .id(file.getId())
            .document(doc)
        );
    }
    
    private boolean isTextExtractable(String mimeType) {
        return mimeType.startsWith("text/") ||
               mimeType.equals("application/pdf") ||
               mimeType.contains("document") ||
               mimeType.contains("spreadsheet") ||
               mimeType.contains("presentation");
    }
}
```

---

## 4. Object Storage (S3) Deep Dive

### A) CONCEPT: What is Object Storage?

Object Storage (like AWS S3) is a storage architecture that manages data as objects rather than files in a file system. Each object contains data, metadata, and a unique identifier.

**What problems does S3 solve here?**

1. **Scalability**: Stores petabytes of data
2. **Durability**: 99.999999999% (11 9's) durability
3. **Cost**: Cheaper than block storage for large files
4. **Availability**: 99.99% availability SLA
5. **API**: Simple REST API for access

### B) OUR USAGE: How We Use S3 Here

**Storage Architecture:**

```
Bucket: file-storage-prod
├── blocks/          (deduplicated blocks)
│   ├── aa/         (first 2 chars of hash)
│   │   ├── aabbcc... (full hash)
│   │   └── aadd11... (full hash)
│   └── bb/
├── metadata/        (file metadata)
└── thumbnails/      (preview images)
```

**Access Patterns:**

- Write: Multipart upload for large files
- Read: Direct GET requests
- Delete: Soft delete (mark as deleted, physical delete later)

### C) REAL STEP-BY-STEP SIMULATION: S3 Technology-Level

**Normal Flow: File Upload to S3**

```
Step 1: Initiate Multipart Upload
┌─────────────────────────────────────────────────────────────┐
│ Client: POST /upload/session                                 │
│ File: 100 MB presentation                                    │
│ S3: CreateMultipartUpload                                    │
│   Bucket: file-storage-prod                                 │
│   Key: blocks/aa/aabbccddee...                              │
│   Returns: upload_id = "xyz123"                            │
│ Latency: ~50ms                                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Upload Parts (25 parts × 4 MB)
┌─────────────────────────────────────────────────────────────┐
│ For each part (1-25):                                        │
│   Client: PUT /upload/session/{id}/chunk/{n}               │
│   S3: UploadPart                                             │
│     UploadId: xyz123                                         │
│     PartNumber: n                                            │
│     Body: 4 MB chunk                                         │
│     Returns: ETag = "etag_n"                                │
│   Latency: ~200ms per part                                   │
│   Total: 25 × 200ms = 5 seconds (parallel upload)          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Complete Multipart Upload
┌─────────────────────────────────────────────────────────────┐
│ Client: POST /upload/session/{id}/complete                  │
│ S3: CompleteMultipartUpload                                 │
│   UploadId: xyz123                                          │
│   Parts: [{PartNumber: 1, ETag: "etag_1"}, ...]           │
│   Returns: Final object key                                 │
│ Latency: ~100ms                                              │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: S3 Upload Failure**

```
Scenario: Network interruption during multipart upload

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Upload starts (25 parts)                              │
│ T+2s: Parts 1-10 uploaded successfully                      │
│ T+3s: Network interruption                                  │
│ T+3s: Parts 11-25 fail                                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+30s: Network recovers                                      │
│ T+31s: Client resumes upload                                 │
│ T+31s: Check uploaded parts: Parts 1-10 already uploaded   │
│ T+32s: Upload remaining parts: 11-25                       │
│ T+37s: All parts uploaded                                   │
│ T+37s: Complete multipart upload                            │
│ T+38s: File available                                        │
└─────────────────────────────────────────────────────────────┘
```

**Idempotency Handling:**

```
Problem: Same part uploaded twice (retry)

Solution: S3 multipart upload is idempotent
1. Same UploadId + PartNumber + ETag → Overwrites (idempotent)
2. CompleteMultipartUpload is idempotent (same parts → same result)
3. Client tracks uploaded parts to avoid re-upload

Example:
- Part 5 uploaded twice → Same ETag → Idempotent
- CompleteMultipartUpload called twice → Same result → Idempotent
```

**Traffic Spike Handling:**

```
Scenario: Millions of files uploaded simultaneously

┌─────────────────────────────────────────────────────────────┐
│ Normal: 1,000 uploads/second                                │
│ Spike: 10,000 uploads/second (viral file sharing)           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ S3 Handling:                                                 │
│ - S3 scales automatically (no capacity limits)            │
│ - Multipart uploads: Parallel parts (no bottleneck)         │
│ - Request rate: 3,500 PUT requests/second per prefix       │
│ - Solution: Distribute across prefixes (hash-based)        │
│                                                              │
│ Result:                                                      │
│ - 10K uploads/second handled by S3                          │
│ - No throttling (distributed prefixes)                      │
│ - All files uploaded successfully                           │
└─────────────────────────────────────────────────────────────┘
```

**Hot Key Mitigation:**

```
Problem: Popular file downloaded by millions

Mitigation:
1. CDN: CloudFront caches popular files at edge
2. Prefix distribution: Hash-based prefixes distribute load
3. Read replicas: S3 automatically handles read scaling

If hot key occurs:
- CDN cache hit rate: 95% (most requests served from edge)
- S3 origin: Handles remaining 5% (distributed across prefixes)
- No S3 throttling
```

**Recovery Behavior:**

```
Auto-healing:
- Upload failure → Retry with exponential backoff → Automatic
- Network timeout → Resume from last part → Automatic
- S3 error → Retry → Automatic

Human intervention:
- Persistent upload failures → Check S3 service status
- High error rate → Scale upload workers or check network
- Storage quota exceeded → Alert and increase quota
```

---

## 4. Object Storage Deep Dive

### Block-Level Deduplication

```java
@Service
public class DeduplicationService {
    
    private static final int BLOCK_SIZE = 4 * 1024 * 1024; // 4 MB
    
    private final BlockRepository blockRepository;
    private final S3Client s3Client;
    
    public UploadResult uploadWithDedup(String userId, InputStream content, long size) {
        List<String> blockHashes = new ArrayList<>();
        List<BlockUpload> blocksToUpload = new ArrayList<>();
        
        byte[] buffer = new byte[BLOCK_SIZE];
        int bytesRead;
        int blockIndex = 0;
        
        while ((bytesRead = content.read(buffer)) != -1) {
            byte[] blockData = Arrays.copyOf(buffer, bytesRead);
            String blockHash = sha256(blockData);
            blockHashes.add(blockHash);
            
            // Check if block already exists
            Optional<Block> existingBlock = blockRepository.findByHash(blockHash);
            
            if (existingBlock.isPresent()) {
                // Block exists - just increment reference count
                blockRepository.incrementRefCount(blockHash);
            } else {
                // New block - need to upload
                blocksToUpload.add(new BlockUpload(blockIndex, blockHash, blockData));
            }
            
            blockIndex++;
        }
        
        // Upload new blocks in parallel
        blocksToUpload.parallelStream().forEach(block -> {
            String storageKey = "blocks/" + block.getHash();
            s3Client.putObject(
                PutObjectRequest.builder()
                    .bucket("file-storage")
                    .key(storageKey)
                    .build(),
                RequestBody.fromBytes(block.getData())
            );
            
            blockRepository.save(new Block(
                block.getHash(),
                block.getData().length,
                storageKey,
                1 // Initial ref count
            ));
        });
        
        // Calculate storage savings
        long totalSize = blockHashes.size() * BLOCK_SIZE;
        long newStorage = blocksToUpload.stream()
            .mapToLong(b -> b.getData().length)
            .sum();
        long savedBytes = totalSize - newStorage;
        
        return new UploadResult(blockHashes, savedBytes);
    }
    
    public void deleteFile(List<String> blockHashes) {
        for (String hash : blockHashes) {
            int newRefCount = blockRepository.decrementRefCount(hash);
            
            if (newRefCount == 0) {
                // No more references - delete block
                Block block = blockRepository.findByHash(hash).orElseThrow();
                s3Client.deleteObject(DeleteObjectRequest.builder()
                    .bucket("file-storage")
                    .key(block.getStorageKey())
                    .build());
                blockRepository.delete(hash);
            }
        }
    }
}
```

### Chunked Upload with Resume

```java
@Service
public class ChunkedUploadService {
    
    private final UploadSessionRepository sessionRepository;
    private final S3Client s3Client;
    
    public UploadSession createSession(CreateSessionRequest request) {
        String sessionId = UUID.randomUUID().toString();
        int chunkSize = 10 * 1024 * 1024; // 10 MB
        int totalChunks = (int) Math.ceil((double) request.getFileSize() / chunkSize);
        
        UploadSession session = UploadSession.builder()
            .id(sessionId)
            .userId(request.getUserId())
            .fileName(request.getFileName())
            .fileSize(request.getFileSize())
            .mimeType(request.getMimeType())
            .parentId(request.getParentId())
            .chunkSize(chunkSize)
            .totalChunks(totalChunks)
            .receivedChunks(0)
            .status(UploadStatus.ACTIVE)
            .expiresAt(Instant.now().plus(Duration.ofHours(24)))
            .build();
        
        sessionRepository.save(session);
        
        // Create S3 multipart upload
        CreateMultipartUploadResponse multipart = s3Client.createMultipartUpload(
            CreateMultipartUploadRequest.builder()
                .bucket("file-storage")
                .key("uploads/" + sessionId)
                .build()
        );
        
        session.setMultipartUploadId(multipart.uploadId());
        sessionRepository.save(session);
        
        return session;
    }
    
    public ChunkResult uploadChunk(String sessionId, int chunkNumber, byte[] data) {
        UploadSession session = sessionRepository.findById(sessionId)
            .orElseThrow(() -> new SessionNotFoundException(sessionId));
        
        if (session.getStatus() != UploadStatus.ACTIVE) {
            throw new SessionExpiredException(sessionId);
        }
        
        // Verify chunk hash
        String chunkHash = sha256(data);
        
        // Upload part to S3
        UploadPartResponse partResponse = s3Client.uploadPart(
            UploadPartRequest.builder()
                .bucket("file-storage")
                .key("uploads/" + sessionId)
                .uploadId(session.getMultipartUploadId())
                .partNumber(chunkNumber + 1) // S3 parts are 1-indexed
                .build(),
            RequestBody.fromBytes(data)
        );
        
        // Record chunk
        UploadChunk chunk = new UploadChunk(
            sessionId,
            chunkNumber,
            chunkHash,
            data.length,
            partResponse.eTag()
        );
        chunkRepository.save(chunk);
        
        // Update session
        session.setReceivedChunks(session.getReceivedChunks() + 1);
        sessionRepository.save(session);
        
        return new ChunkResult(
            chunkNumber,
            data.length,
            session.getReceivedChunks(),
            session.getTotalChunks() - session.getReceivedChunks()
        );
    }
    
    public File completeUpload(String sessionId) {
        UploadSession session = sessionRepository.findById(sessionId)
            .orElseThrow(() -> new SessionNotFoundException(sessionId));
        
        // Verify all chunks received
        if (session.getReceivedChunks() != session.getTotalChunks()) {
            throw new IncompleteUploadException(
                session.getReceivedChunks(), 
                session.getTotalChunks()
            );
        }
        
        // Get all chunk ETags
        List<UploadChunk> chunks = chunkRepository.findBySessionId(sessionId);
        List<CompletedPart> parts = chunks.stream()
            .sorted(Comparator.comparing(UploadChunk::getChunkNumber))
            .map(c -> CompletedPart.builder()
                .partNumber(c.getChunkNumber() + 1)
                .eTag(c.getETag())
                .build())
            .collect(Collectors.toList());
        
        // Complete multipart upload
        s3Client.completeMultipartUpload(
            CompleteMultipartUploadRequest.builder()
                .bucket("file-storage")
                .key("uploads/" + sessionId)
                .uploadId(session.getMultipartUploadId())
                .multipartUpload(CompletedMultipartUpload.builder()
                    .parts(parts)
                    .build())
                .build()
        );
        
        // Move to final location
        String finalKey = "files/" + session.getUserId() + "/" + UUID.randomUUID();
        s3Client.copyObject(CopyObjectRequest.builder()
            .sourceBucket("file-storage")
            .sourceKey("uploads/" + sessionId)
            .destinationBucket("file-storage")
            .destinationKey(finalKey)
            .build());
        
        // Create file record
        File file = fileService.createFile(
            session.getUserId(),
            session.getFileName(),
            session.getParentId(),
            session.getFileSize(),
            session.getMimeType(),
            finalKey,
            calculateFileHash(chunks)
        );
        
        // Cleanup
        session.setStatus(UploadStatus.COMPLETED);
        sessionRepository.save(session);
        s3Client.deleteObject(DeleteObjectRequest.builder()
            .bucket("file-storage")
            .key("uploads/" + sessionId)
            .build());
        
        return file;
    }
}
```

---

## 5. Step-by-Step Simulations

### Simulation 1: File Upload with Deduplication

```
Scenario: User uploads 100 MB presentation that's 80% similar to existing file

Step 1: Client initiates upload
├── Client: POST /upload/session
├── Request: { name: "presentation_v2.pptx", size: 104857600 }
└── Response: { session_id: "sess_123", chunk_size: 4194304, total_chunks: 25 }

Step 2: Client uploads chunks
├── For each 4 MB chunk:
│   ├── Calculate SHA-256 hash
│   ├── PUT /upload/session/sess_123/chunk/{n}
│   └── Server checks if block exists

Step 3: Deduplication check
├── Total chunks: 25
├── Existing blocks found: 20 (80% dedup)
├── New blocks to store: 5
└── Storage used: 20 MB instead of 100 MB

Step 4: Complete upload
├── Client: POST /upload/session/sess_123/complete
├── Server assembles file metadata
├── Server creates file record with block references
└── Response: { file_id: "file_xyz", size: 104857600 }

Step 5: Post-processing
├── Kafka event: FILE_CREATED
├── Search indexer extracts text
├── Thumbnail generator creates preview
└── Quota updated: +100 MB (logical size)

Result:
├── File accessible immediately
├── Only 20 MB new storage used
├── 80 MB saved through deduplication
└── All devices notified within 5 seconds
```

### Simulation 2: Sync Conflict Resolution

```
Scenario: Same file edited on laptop and phone simultaneously

Timeline:
├── T0: File "report.docx" version 5 on all devices
├── T1: Laptop goes offline, user edits file
├── T2: Phone user edits same file (online)
├── T3: Phone sync: version 6 created on server
├── T4: Laptop comes online, tries to sync

Step 1: Laptop detects local change
├── File modified: report.docx
├── Local version: 5 (with edits)
├── Last known server version: 5
└── Action: Upload local changes

Step 2: Laptop uploads changes
├── POST /sync/commit
├── Request: { path: "/report.docx", base_rev: "v5", content_hash: "abc123" }
└── Server detects conflict!

Step 3: Server conflict detection
├── Current server version: 6 (from phone)
├── Client base version: 5
├── Conflict type: BOTH_MODIFIED
└── Response: { status: "conflict", server_version: 6 }

Step 4: Conflict resolution
├── Strategy: Keep both versions
├── Server version: report.docx (v6)
├── Laptop version: report (conflicted copy - Laptop - 2024-01-15).docx
└── Both files now exist

Step 5: Laptop receives resolution
├── Download server version (v6)
├── Rename local version with conflict suffix
├── Upload renamed version
└── User sees both files, can manually merge

Result:
├── No data loss
├── Both edits preserved
├── User notified of conflict
└── Can compare and merge manually
```

---

## 6. Database Transactions Deep Dive

### A) CONCEPT: What are Database Transactions in File Storage Context?

Database transactions ensure ACID properties for critical operations like file creation, metadata updates, and quota management. In a file storage system, transactions are essential for maintaining data consistency when multiple operations must succeed or fail together.

**What problems do transactions solve here?**

1. **Atomicity**: File creation must create file record, update quota, and create block references atomically
2. **Consistency**: Quota limits can't be exceeded, file metadata must be consistent
3. **Isolation**: Concurrent file operations don't interfere with each other
4. **Durability**: Once committed, file metadata persists even if system crashes

**Transaction Isolation Levels:**

| Level | Description | Use Case |
|-------|-------------|----------|
| READ COMMITTED | Default, prevents dirty reads | File listing, metadata reads |
| REPEATABLE READ | Prevents non-repeatable reads | Quota checks, file operations |
| SERIALIZABLE | Highest isolation, prevents phantom reads | Critical quota updates |

### B) OUR USAGE: How We Use Transactions Here

**Critical Transactional Operations:**

1. **File Creation Transaction**:
   - Create file record
   - Update user quota (increment used space)
   - Create block references
   - Update folder metadata
   - All must succeed or all rollback

2. **File Deletion Transaction**:
   - Delete file record
   - Update user quota (decrement used space)
   - Decrement block reference counts
   - Update folder metadata
   - Delete blocks if ref_count reaches 0

3. **Quota Update Transaction**:
   - Check current quota usage
   - Verify quota limit not exceeded
   - Update quota usage
   - Create quota event record

**Transaction Configuration:**

```java
@Transactional(
    isolation = Isolation.REPEATABLE_READ,
    propagation = Propagation.REQUIRED,
    timeout = 30,
    rollbackFor = {Exception.class}
)
public File createFile(CreateFileRequest request) {
    // Transactional operations
}
```

### C) REAL STEP-BY-STEP SIMULATION: Database Transaction Flow

**Normal Flow: File Creation Transaction**

```
Step 1: Begin Transaction
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: BEGIN TRANSACTION                                │
│ Isolation Level: REPEATABLE READ                             │
│ Transaction ID: tx_file_789                                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Quota
┌─────────────────────────────────────────────────────────────┐
│ SELECT quota_limit, quota_used FROM user_quotas             │
│ WHERE user_id = 'user_123'                                     │
│ FOR UPDATE  -- Row-level lock                                │
│                                                              │
│ Result: quota_limit = 10737418240 (10GB),                    │
│         quota_used = 5368709120 (5GB)                        │
│ Lock acquired on quota row                                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Verify Quota Not Exceeded
┌─────────────────────────────────────────────────────────────┐
│ Check: quota_used + file_size <= quota_limit                 │
│        5GB + 100MB = 5.1GB <= 10GB ✅                        │
│                                                              │
│ Quota check passed                                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Create File Record
┌─────────────────────────────────────────────────────────────┐
│ INSERT INTO files (id, user_id, name, size, folder_id, ...) │
│ VALUES ('file_456', 'user_123', 'document.pdf',              │
│         104857600, 'folder_789', ...)                        │
│                                                              │
│ Result: 1 row inserted                                       │
│ File ID: file_456                                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Create Block References
┌─────────────────────────────────────────────────────────────┐
│ INSERT INTO file_blocks (file_id, block_hash, sequence, ...)│
│ VALUES                                                       │
│   ('file_456', 'hash_1', 0, ...),                            │
│   ('file_456', 'hash_2', 1, ...),                            │
│   ('file_456', 'hash_3', 2, ...)                             │
│                                                              │
│ Result: 3 rows inserted                                      │
│ Block references created                                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 6: Update Block Reference Counts
┌─────────────────────────────────────────────────────────────┐
│ UPDATE blocks                                                │
│ SET ref_count = ref_count + 1                                │
│ WHERE hash IN ('hash_1', 'hash_2', 'hash_3')                │
│                                                              │
│ Result: 3 rows updated                                       │
│ Reference counts incremented                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 7: Update User Quota
┌─────────────────────────────────────────────────────────────┐
│ UPDATE user_quotas                                           │
│ SET quota_used = quota_used + 104857600,                     │
│     updated_at = NOW()                                       │
│ WHERE user_id = 'user_123'                                    │
│                                                              │
│ Result: 1 row updated                                       │
│ Quota updated: 5GB → 5.1GB                                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 8: Update Folder Metadata
┌─────────────────────────────────────────────────────────────┐
│ UPDATE folders                                               │
│ SET file_count = file_count + 1,                             │
│     total_size = total_size + 104857600,                    │
│     updated_at = NOW()                                       │
│ WHERE id = 'folder_789'                                      │
│                                                              │
│ Result: 1 row updated                                       │
│ Folder metadata updated                                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 9: Commit Transaction
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: COMMIT TRANSACTION                              │
│                                                              │
│ All changes made permanent:                                 │
│ - File record created                                       │
│ - Block references created                                  │
│ - Block ref_counts updated                                  │
│ - User quota updated                                        │
│ - Folder metadata updated                                   │
│ - Locks released                                            │
│                                                              │
│ Total time: ~60ms                                           │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Quota Exceeded**

```
Step 1: User Tries to Upload File Exceeding Quota
┌─────────────────────────────────────────────────────────────┐
│ Transaction: File creation in progress                      │
│ - File size: 6GB                                             │
│ - Current quota used: 5GB                                   │
│ - Quota limit: 10GB                                          │
│ - Check: 5GB + 6GB = 11GB > 10GB ❌                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Quota Check Fails
┌─────────────────────────────────────────────────────────────┐
│ Application: QuotaExceededException thrown                  │
│                                                              │
│ Exception: "Quota limit exceeded.                            │
│            Available: 5GB, Required: 6GB"                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Automatic Rollback
┌─────────────────────────────────────────────────────────────┐
│ @Transactional detects exception:                           │
│ - Rolls back entire transaction                              │
│ - No file record created                                    │
│ - Quota not updated                                         │
│ - Block references not created                              │
│ - All locks released                                        │
│                                                              │
│ PostgreSQL: ROLLBACK TRANSACTION                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Application Handling
┌─────────────────────────────────────────────────────────────┐
│ Application catches QuotaExceededException:                 │
│                                                              │
│ 1. Return 413 Payload Too Large to client                  │
│ 2. Include quota information in error response             │
│ 3. Suggest upgrade or delete files                          │
│ 4. Log quota violation for analytics                        │
│                                                              │
│ No file created (atomicity maintained)                     │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Concurrent File Deletions**

```
Step 1: Two Users Delete Same File Simultaneously
┌─────────────────────────────────────────────────────────────┐
│ User A deletes file_123                                       │
│ User B deletes file_123 (admin action)                       │
│ Both transactions start simultaneously                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Both Acquire Locks
┌─────────────────────────────────────────────────────────────┐
│ Transaction A: SELECT * FROM files                            │
│                WHERE id = 'file_123' FOR UPDATE              │
│                Lock acquired                                │
│                                                              │
│ Transaction B: SELECT * FROM files                          │
│                WHERE id = 'file_123' FOR UPDATE              │
│                Waiting for lock (blocked)                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Transaction A Deletes File
┌─────────────────────────────────────────────────────────────┐
│ Transaction A:                                                │
│ - DELETE FROM files WHERE id = 'file_123'                    │
│ - UPDATE user_quotas SET quota_used = quota_used - size      │
│ - UPDATE blocks SET ref_count = ref_count - 1                │
│ - COMMIT                                                     │
│                                                              │
│ File deleted, locks released                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Transaction B Continues
┌─────────────────────────────────────────────────────────────┐
│ Transaction B: Lock acquired                                │
│                SELECT * FROM files                           │
│                WHERE id = 'file_123' FOR UPDATE               │
│                Result: 0 rows (file already deleted)         │
│                                                              │
│ Application: FileNotFoundException                          │
│                                                              │
│ Transaction B: ROLLBACK (no changes to rollback)            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Application Handling
┌─────────────────────────────────────────────────────────────┐
│ Application catches FileNotFoundException:                  │
│                                                              │
│ 1. Return 404 Not Found to User B                           │
│ 2. Log idempotent delete (file already deleted)             │
│ 3. No error (idempotent operation)                          │
│                                                              │
│ ✅ Both operations handled correctly                        │
└─────────────────────────────────────────────────────────────┘
```

**Isolation Level Behavior: Quota Updates**

```
Scenario: User uploads file while quota is being updated

Transaction A (Upload):                    Transaction B (Quota Update):
─────────────────────────────────────────────────────────────────
BEGIN TRANSACTION                          BEGIN TRANSACTION
                                           
SELECT quota_used FROM user_quotas        UPDATE user_quotas
WHERE user_id = 'user_123'                 SET quota_limit = 20GB
FOR UPDATE                                  WHERE user_id = 'user_123'
                                           
Result: quota_used = 5GB                   Result: quota_limit = 20GB
        quota_limit = 10GB                 
                                           
Check: 5GB + 1GB = 6GB <= 10GB ✅          COMMIT
                                           
INSERT INTO files (...)                    -- Quota limit updated
                                           
UPDATE user_quotas                         
SET quota_used = 6GB                       
                                           
COMMIT                                     
                                           
✅ File uploaded successfully              ✅ Quota limit increased
                                           
                                           
With REPEATABLE READ:                       With READ COMMITTED:
- Upload sees consistent snapshot          - Upload sees latest quota
  (quota_limit = 10GB)                      (quota_limit = 20GB)
- Uses 10GB limit for check                 - Uses 20GB limit for check
- File uploaded (within old limit)         - File uploaded (within new limit)
- Both operations succeed                  - Both operations succeed
```

**Lock Contention: High-Concurrency File Operations**

```
Step 1: Multiple Users Upload Files Simultaneously
┌─────────────────────────────────────────────────────────────┐
│ 100 concurrent transactions trying to create files           │
│ Many users updating quota simultaneously                     │
│                                                              │
│ Lock queue forms on user_quotas rows                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Lock Acquisition Strategy
┌─────────────────────────────────────────────────────────────┐
│ Application: Acquire locks in consistent order              │
│ 1. Lock user quota row (by user_id)                         │
│ 2. Lock folder row (by folder_id)                           │
│ 3. Create file record                                       │
│ 4. Update block ref_counts                                  │
│                                                              │
│ Prevents deadlocks by consistent lock ordering             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Optimistic Locking for Quota Updates
┌─────────────────────────────────────────────────────────────┐
│ Instead of row locks, use version-based optimistic locking:│
│                                                              │
│ UPDATE user_quotas                                          │
│ SET quota_used = quota_used + :file_size,                    │
│     version = version + 1                                    │
│ WHERE user_id = :user_id                                    │
│   AND version = :expected_version                           │
│   AND quota_used + :file_size <= quota_limit                │
│                                                              │
│ If version mismatch: Retry with new version                 │
│ Reduces lock contention                                     │
└─────────────────────────────────────────────────────────────┘
```

**Retry Logic for Failed Transactions**

```java
@Retryable(
    value = {DeadlockLoserDataAccessException.class,
             OptimisticLockingFailureException.class,
             TransactionTimeoutException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 200, multiplier = 2)
)
@Transactional(isolation = Isolation.REPEATABLE_READ)
public File createFile(CreateFileRequest request) {
    // Transaction logic
    // Retries automatically on deadlock/optimistic lock failure
}
```

**Recovery Behavior:**

```
Transaction Failure → Automatic Retry:
1. Deadlock detected → Retry with 200ms delay
2. Optimistic lock failure → Retry with 400ms delay
3. Quota exceeded → No retry (business logic error)
4. File not found → No retry (business logic error)
5. Max retries exceeded → Return error to client

Monitoring:
- Track retry rate (alert if > 2%)
- Track deadlock rate (alert if > 0.5%)
- Track optimistic lock failure rate (alert if > 1%)
- Track quota violation rate (alert if > 5%)
```

---

