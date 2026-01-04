# File Storage System - Interview Grilling Q&A

## Trade-off Questions

### Q1: Why block-level deduplication instead of file-level?

**Answer:**

```
Block-level deduplication is chosen because:

1. Higher Dedup Ratio:
   - File-level: Only exact duplicates saved
   - Block-level: Similar files share common blocks
   - Example: 100 MB file with 1 KB change
     - File-level: 200 MB stored (both versions)
     - Block-level: 104 MB stored (100 MB + 4 MB new block)

2. Version History Efficiency:
   - Each version shares unchanged blocks
   - Only deltas stored
   - 10 versions of 100 MB file:
     - File-level: 1 TB
     - Block-level: ~150 MB (if 5% changes per version)

3. Cross-User Dedup:
   - Popular files (OS images, software) shared
   - Legal documents, templates deduplicated
   - Savings: 20-30% additional

Trade-offs:
├── More complex implementation
├── CPU overhead for hashing
├── Reference counting for cleanup
└── Block size tuning required

When would I use file-level?
- Simpler implementation needed
- Files rarely modified (archive storage)
- Compute resources limited
```

### Q2: Why separate metadata and content storage?

**Answer:**

```
Separation provides:

1. Different Access Patterns:
   - Metadata: Small, frequent, transactional
   - Content: Large, less frequent, streaming
   - PostgreSQL optimized for metadata
   - S3 optimized for blob storage

2. Independent Scaling:
   - Metadata: Scale by adding shards
   - Content: Infinite scale with S3
   - No coupling between them

3. Cost Optimization:
   - Metadata: Fast SSD storage (~$0.10/GB)
   - Content: Tiered object storage (~$0.01/GB)
   - 10x cost difference

4. Durability Guarantees:
   - S3: 11 nines durability built-in
   - Would need complex replication otherwise

5. Operational Simplicity:
   - S3 is managed, no ops overhead
   - Focus engineering on metadata layer

Alternative: Unified storage (like HDFS)
├── Simpler architecture
├── Better for analytics workloads
├── But: Higher ops burden, less durable
└── Not recommended for user-facing storage
```

### Q3: How do you handle the sync cursor for millions of users?

**Answer:**

```
Cursor design:

1. Cursor Content:
   - User ID
   - Device ID
   - Last event ID (monotonic)
   - Path prefix (for selective sync)
   - Timestamp

2. Storage Strategy:
   - Redis for active cursors (24h TTL)
   - PostgreSQL for persistence
   - Lazy loading: Redis miss → load from DB

3. Scalability:
   - 5M concurrent users
   - 1 cursor per device per user
   - ~20M active cursors
   - Redis: 20M × 200 bytes = 4 GB

4. Event ID Design:
   - Monotonically increasing per user
   - Allows efficient "changes since X" queries
   - Index: (user_id, event_id)

Implementation:
```java
public SyncDelta getDelta(String cursor, int limit) {
    SyncCursor parsed = SyncCursor.parse(cursor);
    
    // Query changes since cursor
    List<FileChange> changes = changeRepository
        .findByUserIdAndEventIdGreaterThan(
            parsed.getUserId(),
            parsed.getLastEventId(),
            PageRequest.of(0, limit + 1)
        );
    
    // Build new cursor
    long newEventId = changes.isEmpty() 
        ? parsed.getLastEventId()
        : changes.get(changes.size() - 1).getEventId();
    
    return new SyncDelta(
        changes.subList(0, Math.min(changes.size(), limit)),
        new SyncCursor(parsed.getUserId(), newEventId).encode(),
        changes.size() > limit
    );
}
```
```

### Q4: Why use WebSocket for sync notifications instead of polling?

**Answer:**

```
WebSocket advantages:

1. Efficiency:
   - Polling: 5M users × 2 req/min = 167K req/sec
   - WebSocket: Only actual changes pushed
   - 99% reduction in unnecessary requests

2. Latency:
   - Polling: Up to 30 sec delay
   - WebSocket: < 1 second push
   - Better user experience

3. Battery/Data (Mobile):
   - Polling: Constant wake-ups
   - WebSocket: Single persistent connection
   - 80% battery savings on mobile

Trade-offs:
├── Connection management complexity
├── Sticky sessions needed
├── Harder to scale (stateful)
└── Fallback needed for restrictive networks

Hybrid approach:
1. WebSocket for connected devices
2. Push notifications for mobile (FCM/APNS)
3. Long-polling fallback for restricted networks
4. Full delta sync on reconnection

Scale handling:
- 5M concurrent WebSocket connections
- 50K connections per server
- 100 WebSocket servers
- HAProxy for connection routing
```

---

## Scaling Questions

### Q5: How would you handle 10x growth in one year?

**Answer:**

```
Current: 500 PB storage, 50M DAU
Target: 5 EB storage, 500M DAU

Phase 1: Immediate Scaling (Month 1-3)
├── Add compute capacity (auto-scaling handles)
├── Increase database shards: 100 → 300
├── Scale Redis cluster: 200 → 500 nodes
├── Add Elasticsearch nodes: 113 → 300
└── Increase Kafka partitions

Phase 2: Architecture Evolution (Month 4-6)
├── Multi-region deployment (currently single)
├── Regional data residency
├── Edge caching for popular files
└── Async processing for non-critical paths

Phase 3: Optimization (Month 7-12)
├── Improved deduplication algorithms
├── Smarter storage tiering
├── Query optimization
└── Cost reduction initiatives

Key Challenges:
1. Database migration (online resharding)
2. Maintaining consistency during growth
3. Cost management (10x storage cost)
4. Team scaling (can't 10x engineers)

Cost projection:
├── Current: $10M/month
├── Linear scale: $100M/month
├── With optimization: $60M/month
└── Target: < $0.15/user/month
```

### Q6: What breaks first under load?

**Answer:**

```
Failure order during traffic spike:

1. Sync Service (First to fail)
   - Symptom: WebSocket connections dropping
   - Cause: Connection limit per server
   - Impact: Devices stop receiving updates
   - Fix: Scale WebSocket servers, increase limits

2. Database Connections
   - Symptom: Connection pool exhausted
   - Cause: Too many concurrent requests
   - Impact: API errors, timeouts
   - Fix: PgBouncer, increase pool, add replicas

3. Upload Service
   - Symptom: Upload timeouts
   - Cause: S3 rate limiting or network saturation
   - Impact: Users can't upload
   - Fix: Request increase from S3, add upload servers

4. Search Index
   - Symptom: Slow searches, timeouts
   - Cause: Elasticsearch overwhelmed
   - Impact: Users can't find files
   - Fix: Add ES nodes, reduce indexing rate

5. CDN
   - Symptom: Download failures
   - Cause: Origin overload
   - Impact: Users can't download
   - Fix: Increase cache TTL, add origin capacity

Mitigation: Load shedding
- Prioritize core operations (upload, download)
- Disable non-essential features (suggestions, analytics)
- Queue background processing
```

### Q7: How do you handle a user with 10 million files?

**Answer:**

```
Challenge: Power users with massive file counts

Problems:
├── Folder listing takes forever
├── Sync delta is huge
├── Search is slow
└── Quota calculation expensive

Solutions:

1. Pagination Everywhere:
   - Never return all files at once
   - Cursor-based pagination
   - Limit: 1000 items per request

2. Incremental Sync:
   - Never full sync for large accounts
   - Always delta from last cursor
   - Batch size: 500 changes max

3. Folder Caching:
   - Cache folder item counts
   - Invalidate on change
   - Don't recalculate on every request

4. Async Quota:
   - Don't calculate quota synchronously
   - Background job updates quota
   - Eventual consistency acceptable

5. Search Optimization:
   - Per-user search index partition
   - Limit search scope by default
   - "Search in folder" option

6. Selective Sync:
   - Don't sync entire account
   - User chooses folders to sync
   - Desktop client handles locally

Implementation:
```java
// Folder listing with streaming
public Flux<FileItem> streamFolderContents(String folderId) {
    return Flux.create(sink -> {
        String cursor = null;
        do {
            Page<FileItem> page = getPage(folderId, cursor, 100);
            page.getItems().forEach(sink::next);
            cursor = page.getNextCursor();
        } while (cursor != null);
        sink.complete();
    });
}
```
```

---

## Failure Scenarios

### Q8: What happens if S3 becomes unavailable?

**Answer:**

```
S3 failure is catastrophic but handled:

Immediate Impact:
├── New uploads fail
├── Downloads fail (for non-cached files)
├── Thumbnails unavailable
└── Metadata operations still work

Detection:
├── Health checks fail
├── Error rate spikes
├── Circuit breaker opens
└── Alert triggered

Mitigation Layers:

1. CDN Cache (First defense):
   - Popular files served from CDN
   - 60% of downloads unaffected
   - Extended cache TTL during outage

2. Cross-Region Replication:
   - Files replicated to DR region
   - Failover to DR S3
   - RTO: 5-10 minutes

3. Graceful Degradation:
   - Uploads queued locally
   - "Upload pending" status shown
   - Retry when S3 recovers

4. User Communication:
   - Status page updated
   - In-app notification
   - "Service degraded" banner

Recovery:
├── S3 recovers (usually < 1 hour)
├── Circuit breaker closes
├── Queued uploads processed
├── Normal operation resumes

Historical S3 outages:
├── 2017: 4 hours (US-East-1)
├── Since then: < 10 minutes typically
└── 99.99% availability SLA
```

### Q9: How do you handle ransomware attacks?

**Answer:**

```
Ransomware encrypts user files maliciously:

Detection:
├── Unusual file modification patterns
├── Mass file renames (.encrypted extension)
├── Rapid version creation
├── ML model flags suspicious activity

Prevention:
1. Version History:
   - Keep 180 days of versions
   - Ransomware can't delete versions
   - User can restore pre-attack state

2. Suspicious Activity Alerts:
   - "1000 files modified in 1 minute"
   - Notify user immediately
   - Option to pause sync

3. Restore Points:
   - Daily account snapshots
   - One-click restore to point in time
   - Independent of file versions

Response:
1. Detect attack (automated or user report)
2. Pause sync for affected devices
3. Identify attack start time
4. Restore all files to pre-attack state
5. Revoke compromised device access
6. Guide user on malware removal

Implementation:
```java
@Scheduled(fixedRate = 60000)
public void detectRansomware() {
    // Find users with suspicious activity
    List<UserActivity> suspicious = activityRepository
        .findUsersWithHighModificationRate(
            threshold: 100,  // files per minute
            window: Duration.ofMinutes(5)
        );
    
    for (UserActivity activity : suspicious) {
        // Check for ransomware patterns
        if (hasRansomwarePattern(activity)) {
            // Pause sync and alert
            syncService.pauseUser(activity.getUserId());
            alertService.sendRansomwareAlert(activity.getUserId());
        }
    }
}
```
```

### Q10: What if a sync conflict corrupts data?

**Answer:**

```
Sync conflicts should never corrupt data:

Design Principles:
1. Never overwrite without backup
2. Keep both versions on conflict
3. User decides resolution
4. Audit trail of all changes

Conflict Types:

1. Both Modified:
   - Keep server version as-is
   - Save client version with conflict suffix
   - User sees both, can merge manually

2. Delete vs Modify:
   - Modified version wins
   - Deleted file restored
   - User notified

3. Move vs Modify:
   - Both changes applied
   - File moved AND updated
   - No conflict

4. Rename Collision:
   - Second file gets suffix
   - "file.txt" and "file (1).txt"

Recovery from corruption:
1. Version history available
2. Restore any previous version
3. Account-level restore point
4. Support can access deleted files

Safeguards:
```java
public SyncResult applyChange(FileChange change) {
    File serverFile = fileRepository.findById(change.getFileId());
    
    // Never delete without checking
    if (change.getType() == DELETE) {
        if (serverFile.getModifiedAt().isAfter(change.getBaseTimestamp())) {
            // File modified after client's view - conflict
            return SyncResult.conflict(
                ConflictType.DELETE_MODIFIED,
                serverFile
            );
        }
        // Safe to delete - move to trash
        trashService.moveToTrash(serverFile);
        return SyncResult.success();
    }
    
    // Always create version before overwrite
    versionService.createVersion(serverFile);
    
    // Apply change
    serverFile.setContent(change.getContent());
    fileRepository.save(serverFile);
    
    return SyncResult.success();
}
```
```

---

## Design Evolution Questions

### Q11: What would you build first with 4 engineers and 8 weeks?

**Answer:**

```
MVP Scope (8 weeks):

Week 1-2: Core Infrastructure
├── User authentication (OAuth)
├── PostgreSQL schema
├── S3 integration
├── Basic API structure

Week 3-4: File Operations
├── File upload (simple, no chunking)
├── File download
├── Folder CRUD
├── File listing

Week 5-6: Basic Sync
├── Delta sync API
├── Conflict detection (simple)
├── Desktop client integration
└── Manual sync trigger

Week 7-8: Sharing & Polish
├── Share links
├── Basic permissions
├── Error handling
├── Documentation

What's NOT included:
├── Chunked uploads (< 100 MB limit)
├── Deduplication
├── Version history
├── Search
├── Real-time sync (polling only)
├── Mobile apps
├── Team features

This MVP:
├── Handles 1000 users
├── 10 GB per user
├── Basic but functional
├── Foundation for growth
```

### Q12: How does the architecture evolve over 5 years?

**Answer:**

```
Year 1: MVP to Product
├── 10K users, 100 TB storage
├── Single region
├── Basic features
├── Manual operations
└── Cost: $10K/month

Year 2: Growth
├── 1M users, 10 PB storage
├── Add search (Elasticsearch)
├── Add version history
├── Real-time sync (WebSocket)
├── Mobile apps
├── Chunked uploads
└── Cost: $200K/month

Year 3: Scale
├── 50M users, 100 PB storage
├── Multi-region deployment
├── Block-level deduplication
├── Storage tiering
├── Team features
├── Enterprise SSO
└── Cost: $2M/month

Year 4: Enterprise
├── 200M users, 300 PB storage
├── Compliance (HIPAA, SOC2)
├── Advanced security
├── Admin console
├── API platform
├── Third-party integrations
└── Cost: $6M/month

Year 5: Platform
├── 500M users, 500 PB storage
├── AI-powered features
├── Workflow automation
├── Developer ecosystem
├── Global presence
├── Acquisition targets
└── Cost: $10M/month

Key Architecture Changes:
├── Year 1→2: Add caching, search
├── Year 2→3: Sharding, multi-region
├── Year 3→4: Compliance, security
├── Year 4→5: Platform, extensibility
```

---

## Level-Specific Expectations

### L4 (Entry-Level) Expectations

Should demonstrate:
- Understanding of client-server file operations
- Basic database schema design
- Awareness of object storage (S3)
- Simple sync approach

May struggle with:
- Deduplication strategies
- Conflict resolution details
- Scale considerations
- Security implementation

### L5 (Mid-Level) Expectations

Should demonstrate:
- Complete sync algorithm
- Caching strategy
- Event-driven architecture
- Failure handling
- Monitoring approach

May struggle with:
- Multi-region deployment
- Cost optimization at scale
- Complex security (encryption, compliance)
- Large-scale operational concerns

### L6 (Senior) Expectations

Should demonstrate:
- Multiple architecture approaches
- Deep dive into any component
- Cost analysis and optimization
- Security and compliance
- Organizational considerations
- Evolution over time

---

## Common Interviewer Pushbacks

### "Why not just use a single database?"

**Response:**

```
I considered single database, but separated because:

1. Different Data Characteristics:
   - Metadata: Structured, small, transactional
   - Content: Unstructured, large, streaming
   - One database can't optimize for both

2. Scale Requirements:
   - 500 PB of content
   - No relational database handles this
   - S3 designed for this scale

3. Cost:
   - Database storage: $0.10-0.20/GB
   - S3 storage: $0.01-0.02/GB
   - 10x cost difference at scale

4. Durability:
   - S3: 11 nines durability
   - Self-managed: Hard to achieve

For smaller scale (< 1 TB):
- Single database could work
- Simpler architecture
- Lower operational complexity
```

### "How do you ensure files are never lost?"

**Response:**

```
Multiple layers of protection:

1. Object Storage Durability:
   - S3: 99.999999999% durability
   - 3 copies within region
   - 2 copies in DR region

2. Write Confirmation:
   - Upload not confirmed until S3 acknowledges
   - Client retries on failure
   - No "fire and forget"

3. Metadata Consistency:
   - File record created AFTER content stored
   - Orphan content cleaned up, not orphan metadata
   - Never reference non-existent content

4. Version History:
   - Old versions preserved
   - Accidental overwrites recoverable
   - 180 days retention

5. Trash:
   - Deleted files in trash 30 days
   - Can be restored

6. Backups:
   - Metadata: Point-in-time recovery
   - Daily snapshots
   - Cross-region replication

Failure modes handled:
├── Client crash: Retry upload
├── Network failure: Resume from chunk
├── S3 failure: Failover to DR
├── Database failure: Restore from replica
├── User error: Version history / trash
```

### "This seems over-engineered for file storage"

**Response:**

```
The complexity matches the requirements:

At Dropbox scale (500M users, 500 PB):
├── 200K requests/second
├── 5 TB uploaded per hour
├── 99.99% availability required
├── Zero data loss tolerance

Complexity drivers:
1. Sync across devices (hardest problem)
2. Deduplication (cost requirement)
3. Version history (user expectation)
4. Security (enterprise requirement)
5. Scale (can't do with simple architecture)

For smaller scale:
├── 10K users: Single server, local storage
├── 100K users: Add S3, basic sync
├── 1M users: Add caching, search
├── 10M users: Sharding, multi-region

I can simplify based on actual requirements.
The architecture I described handles Dropbox-scale.
```

---

## Summary: Key Points to Remember

1. **Separation of concerns**: Metadata (PostgreSQL) vs Content (S3)
2. **Block-level deduplication**: 30% storage savings
3. **Cursor-based sync**: Efficient delta synchronization
4. **Storage tiering**: Hot/Warm/Cold for cost optimization
5. **Conflict resolution**: Never lose data, keep both versions
6. **Encryption**: Per-file keys, KMS for key management
7. **Multi-region**: DR and data residency
8. **Cost awareness**: $0.02/GB/month target
9. **Graceful degradation**: Core features always work

