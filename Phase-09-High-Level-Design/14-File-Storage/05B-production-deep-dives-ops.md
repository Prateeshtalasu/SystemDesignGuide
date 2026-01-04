# File Storage System - Production Deep Dives (Operations)

## 1. Scaling Strategy

### Horizontal Scaling Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Scaling Architecture                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Stateless Services (Auto-scale based on CPU/requests):                       │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  API Gateway        │ 20-100 pods │ Scale on requests/sec          │     │
│  │  Metadata Service   │ 50-200 pods │ Scale on CPU                   │     │
│  │  Upload Service     │ 50-500 pods │ Scale on concurrent uploads    │     │
│  │  Download Service   │ 50-500 pods │ Scale on bandwidth             │     │
│  │  Sync Service       │ 20-100 pods │ Scale on WebSocket connections │     │
│  │  Search Service     │ 20-50 pods  │ Scale on query latency         │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  Stateful Services (Manual scaling with planning):                            │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  PostgreSQL         │ 100 shards  │ Add shards for capacity        │     │
│  │  Redis Cluster      │ 200 nodes   │ Add nodes for memory           │     │
│  │  Elasticsearch      │ 113 nodes   │ Add nodes for index size       │     │
│  │  Kafka              │ 30 brokers  │ Add partitions for throughput  │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  Object Storage (Managed, infinite scale):                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  S3 / GCS           │ 500 PB      │ Automatic scaling              │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Database Sharding Scale-Out

```java
// Adding new shards without downtime
public class ShardManager {
    
    // Current: 100 shards
    // Target: 150 shards (50% increase)
    
    public void addShards(int newShardCount) {
        // Phase 1: Add new shard nodes (empty)
        for (int i = currentShardCount; i < newShardCount; i++) {
            provisionNewShard(i);
        }
        
        // Phase 2: Enable dual-write to old and new shards
        enableDualWrite(newShardCount);
        
        // Phase 3: Background migration of existing data
        migrateData(currentShardCount, newShardCount);
        
        // Phase 4: Switch reads to new shard mapping
        updateShardMapping(newShardCount);
        
        // Phase 5: Disable dual-write, cleanup old data
        disableDualWrite();
        cleanupMigratedData();
    }
    
    private int getShardForUser(String userId, int shardCount) {
        // Consistent hashing for minimal data movement
        return Math.abs(userId.hashCode()) % shardCount;
    }
}
```

### Auto-Scaling Configuration

```yaml
# Kubernetes HPA for Upload Service
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: upload-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: upload-service
  minReplicas: 50
  maxReplicas: 500
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Pods
    pods:
      metric:
        name: concurrent_uploads
      target:
        type: AverageValue
        averageValue: 100
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

---

## 2. Reliability Patterns

### Circuit Breaker Implementation

```java
@Service
public class StorageServiceWithCircuitBreaker {
    
    private final CircuitBreaker circuitBreaker;
    private final S3Client primaryS3;
    private final S3Client fallbackS3;  // Different region
    
    public StorageServiceWithCircuitBreaker() {
        this.circuitBreaker = CircuitBreaker.ofDefaults("s3-storage");
        
        circuitBreaker.getEventPublisher()
            .onStateTransition(event -> {
                log.warn("Circuit breaker state: {} -> {}", 
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState());
                alerting.sendAlert("S3 circuit breaker: " + event.getStateTransition());
            });
    }
    
    public byte[] downloadFile(String key) {
        return circuitBreaker.executeSupplier(() -> {
            try {
                return downloadFromPrimary(key);
            } catch (S3Exception e) {
                if (isTransientError(e)) {
                    throw e;  // Let circuit breaker track
                }
                throw new StorageException("Failed to download", e);
            }
        });
    }
    
    @Recover
    public byte[] downloadFileFallback(String key, Exception e) {
        log.warn("Primary S3 unavailable, using fallback for {}", key);
        metrics.increment("storage.fallback.used");
        return downloadFromFallback(key);
    }
}
```

### Retry with Exponential Backoff

```java
@Service
public class RetryableStorageService {
    
    private final RetryTemplate retryTemplate;
    
    public RetryableStorageService() {
        this.retryTemplate = RetryTemplate.builder()
            .maxAttempts(3)
            .exponentialBackoff(100, 2, 5000)  // 100ms, 200ms, 400ms...
            .retryOn(TransientStorageException.class)
            .retryOn(TimeoutException.class)
            .build();
    }
    
    public void uploadWithRetry(String key, byte[] content) {
        retryTemplate.execute(context -> {
            int attempt = context.getRetryCount() + 1;
            log.debug("Upload attempt {} for key {}", attempt, key);
            
            try {
                s3Client.putObject(
                    PutObjectRequest.builder()
                        .bucket("file-storage")
                        .key(key)
                        .build(),
                    RequestBody.fromBytes(content)
                );
                return null;
            } catch (S3Exception e) {
                if (e.statusCode() == 503 || e.statusCode() == 500) {
                    throw new TransientStorageException(e);
                }
                throw e;
            }
        });
    }
}
```

### Graceful Degradation

```java
@Service
public class FileService {
    
    private final FeatureFlags featureFlags;
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    public FileResponse getFile(String fileId) {
        File file = getFileMetadata(fileId);
        FileResponse response = new FileResponse(file);
        
        // Thumbnails - optional, degrade gracefully
        if (featureFlags.isEnabled("thumbnails")) {
            try {
                response.setThumbnailUrl(getThumbnail(file));
            } catch (Exception e) {
                log.warn("Thumbnail unavailable for {}", fileId);
                response.setThumbnailUrl(null);  // Degrade gracefully
            }
        }
        
        // Search suggestions - optional
        if (featureFlags.isEnabled("suggestions") && 
            !circuitBreakerRegistry.circuitBreaker("elasticsearch").getState().equals(State.OPEN)) {
            try {
                response.setSuggestions(getSuggestions(file));
            } catch (Exception e) {
                log.warn("Suggestions unavailable");
                response.setSuggestions(Collections.emptyList());
            }
        }
        
        // Version history - optional
        if (featureFlags.isEnabled("version-history")) {
            try {
                response.setVersionCount(getVersionCount(file));
            } catch (Exception e) {
                response.setVersionCount(null);
            }
        }
        
        return response;
    }
}
```

---

## 3. Monitoring and Observability

### Key Metrics

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Key Metrics Dashboard                               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Upload Metrics:                                                              │
│  ├── upload_requests_total (counter)                                         │
│  ├── upload_bytes_total (counter)                                            │
│  ├── upload_duration_seconds (histogram)                                     │
│  ├── upload_errors_total (counter by error_type)                            │
│  ├── concurrent_uploads (gauge)                                              │
│  └── dedup_savings_bytes (counter)                                           │
│                                                                               │
│  Download Metrics:                                                            │
│  ├── download_requests_total (counter)                                       │
│  ├── download_bytes_total (counter)                                          │
│  ├── download_duration_seconds (histogram)                                   │
│  ├── cdn_hit_ratio (gauge)                                                   │
│  └── cold_storage_restores (counter)                                         │
│                                                                               │
│  Sync Metrics:                                                                │
│  ├── sync_requests_total (counter)                                           │
│  ├── sync_latency_seconds (histogram)                                        │
│  ├── sync_conflicts_total (counter)                                          │
│  ├── active_websocket_connections (gauge)                                    │
│  └── sync_lag_seconds (histogram)                                            │
│                                                                               │
│  Storage Metrics:                                                             │
│  ├── total_files (gauge)                                                     │
│  ├── total_storage_bytes (gauge)                                             │
│  ├── storage_by_tier (gauge by tier)                                        │
│  └── dedup_ratio (gauge)                                                     │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Alerting Rules

```yaml
# Prometheus alerting rules
groups:
- name: file-storage-alerts
  rules:
  
  # Upload failures
  - alert: HighUploadErrorRate
    expr: |
      rate(upload_errors_total[5m]) / rate(upload_requests_total[5m]) > 0.01
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Upload error rate > 1%"
      
  # Sync latency
  - alert: HighSyncLatency
    expr: |
      histogram_quantile(0.99, rate(sync_latency_seconds_bucket[5m])) > 10
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "P99 sync latency > 10 seconds"
      
  # Storage capacity
  - alert: StorageCapacityWarning
    expr: |
      total_storage_bytes / storage_capacity_bytes > 0.8
    for: 1h
    labels:
      severity: warning
    annotations:
      summary: "Storage usage > 80%"
      
  # Database connections
  - alert: DatabaseConnectionPoolExhausted
    expr: |
      pg_stat_activity_count / pg_settings_max_connections > 0.9
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Database connection pool > 90%"

### Database Connection Pool Configuration

**Connection Pool Settings (HikariCP):**

```yaml
spring:
  datasource:
    hikari:
      # Pool sizing
      maximum-pool-size: 25          # Max connections per application instance
      minimum-idle: 5                 # Minimum idle connections
      
      # Timeouts
      connection-timeout: 30000       # 30s - Max time to wait for connection from pool
      idle-timeout: 600000            # 10m - Idle connection timeout
      max-lifetime: 1800000           # 30m - Max connection lifetime
      
      # Monitoring
      leak-detection-threshold: 60000 # 60s - Detect connection leaks
      
      # Validation
      connection-test-query: SELECT 1
      validation-timeout: 3000        # 3s - Connection validation timeout
      
      # Register metrics
      register-mbeans: true
```

**Pool Sizing Calculation:**

```
Pool size = ((Core count × 2) + Effective spindle count)

For our service (sharded PostgreSQL):
- 4 CPU cores per pod
- Spindle count = 1 per shard
- Calculated: (4 × 2) + 1 = 9 connections minimum
- With safety margin: 25 connections per pod
- 200 pods × 25 = 5000 max connections across all shards
- Per-shard max_connections: 100 (distributed across 100 shards)
```

**Pool Exhaustion Mitigation:**

1. **Monitoring:**
   - Alert when pool usage > 80%
   - Track wait time for connections
   - Monitor connection leak detection

2. **Circuit Breaker:**
   - If pool exhausted for > 5 seconds, open circuit breaker
   - Fail fast instead of blocking

3. **PgBouncer (Connection Pooler):**
   - Place PgBouncer between app and PostgreSQL shards
   - PgBouncer pool: 50 connections per shard
   - App pools can share PgBouncer connections efficiently
      
  # Elasticsearch health
  - alert: ElasticsearchClusterRed
    expr: |
      elasticsearch_cluster_health_status{color="red"} == 1
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Elasticsearch cluster is RED"
```

### Distributed Tracing

```java
@Service
public class TracedUploadService {
    
    private final Tracer tracer;
    
    public File uploadFile(UploadRequest request) {
        Span span = tracer.spanBuilder("upload-file")
            .setAttribute("user.id", request.getUserId())
            .setAttribute("file.name", request.getFileName())
            .setAttribute("file.size", request.getFileSize())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // Deduplication check
            Span dedupSpan = tracer.spanBuilder("check-deduplication")
                .startSpan();
            DedupResult dedupResult = checkDeduplication(request.getContentHash());
            dedupSpan.setAttribute("dedup.found", dedupResult.isFound());
            dedupSpan.end();
            
            // Upload to storage
            Span storageSpan = tracer.spanBuilder("upload-to-storage")
                .startSpan();
            String storageKey = uploadToStorage(request.getContent());
            storageSpan.setAttribute("storage.key", storageKey);
            storageSpan.end();
            
            // Create metadata
            Span metadataSpan = tracer.spanBuilder("create-metadata")
                .startSpan();
            File file = createFileMetadata(request, storageKey);
            metadataSpan.setAttribute("file.id", file.getId());
            metadataSpan.end();
            
            // Publish event
            Span eventSpan = tracer.spanBuilder("publish-event")
                .startSpan();
            publishFileCreatedEvent(file);
            eventSpan.end();
            
            span.setAttribute("result", "success");
            return file;
            
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

---

## 4. Security Deep Dive

### Authentication and Authorization

```java
@Service
public class FileAccessControl {
    
    private final PermissionRepository permissionRepository;
    private final ShareRepository shareRepository;
    private final TeamRepository teamRepository;
    
    public boolean canAccess(String userId, String fileId, Permission required) {
        File file = fileRepository.findById(fileId)
            .orElseThrow(() -> new FileNotFoundException(fileId));
        
        // Owner has full access
        if (file.getOwnerId().equals(userId)) {
            return true;
        }
        
        // Check direct share
        Optional<UserShare> directShare = shareRepository
            .findByFileAndUser(fileId, userId);
        if (directShare.isPresent() && 
            hasPermission(directShare.get().getAccessLevel(), required)) {
            return true;
        }
        
        // Check folder inheritance
        String folderId = file.getParentId();
        while (folderId != null) {
            Optional<UserShare> folderShare = shareRepository
                .findByFolderAndUser(folderId, userId);
            if (folderShare.isPresent() && 
                hasPermission(folderShare.get().getAccessLevel(), required)) {
                return true;
            }
            Folder folder = folderRepository.findById(folderId).orElse(null);
            folderId = folder != null ? folder.getParentId() : null;
        }
        
        // Check team membership
        List<Team> userTeams = teamRepository.findByUserId(userId);
        for (Team team : userTeams) {
            Optional<TeamFolder> teamFolder = teamFolderRepository
                .findByTeamAndContainingFile(team.getId(), fileId);
            if (teamFolder.isPresent() && 
                hasPermission(teamFolder.get().getDefaultAccessLevel(), required)) {
                return true;
            }
        }
        
        return false;
    }
    
    private boolean hasPermission(AccessLevel granted, Permission required) {
        return granted.getPermissions().contains(required);
    }
}
```

### Encryption Key Management

```java
@Service
public class EncryptionService {
    
    private final KmsClient kmsClient;
    private final String masterKeyId;
    
    // Generate Data Encryption Key for new file
    public EncryptionResult encryptFile(byte[] content) {
        // Generate DEK using KMS
        GenerateDataKeyResponse dataKey = kmsClient.generateDataKey(
            GenerateDataKeyRequest.builder()
                .keyId(masterKeyId)
                .keySpec(DataKeySpec.AES_256)
                .build()
        );
        
        // Encrypt content with DEK
        byte[] plainDek = dataKey.plaintext().asByteArray();
        byte[] encryptedContent = encryptWithAes(content, plainDek);
        
        // Clear plain DEK from memory
        Arrays.fill(plainDek, (byte) 0);
        
        return new EncryptionResult(
            encryptedContent,
            dataKey.ciphertextBlob().asByteArray()  // Encrypted DEK
        );
    }
    
    // Decrypt file content
    public byte[] decryptFile(byte[] encryptedContent, byte[] encryptedDek) {
        // Decrypt DEK using KMS
        DecryptResponse decrypted = kmsClient.decrypt(
            DecryptRequest.builder()
                .keyId(masterKeyId)
                .ciphertextBlob(SdkBytes.fromByteArray(encryptedDek))
                .build()
        );
        
        byte[] plainDek = decrypted.plaintext().asByteArray();
        
        // Decrypt content with DEK
        byte[] content = decryptWithAes(encryptedContent, plainDek);
        
        // Clear plain DEK from memory
        Arrays.fill(plainDek, (byte) 0);
        
        return content;
    }
    
    // Rotate encryption keys
    public void rotateKeys(String fileId) {
        File file = fileRepository.findById(fileId).orElseThrow();
        
        // Download and decrypt with old key
        byte[] encryptedContent = storageService.download(file.getStorageKey());
        byte[] content = decryptFile(encryptedContent, file.getEncryptedDek());
        
        // Re-encrypt with new key
        EncryptionResult newEncryption = encryptFile(content);
        
        // Upload re-encrypted content
        String newStorageKey = storageService.upload(newEncryption.getEncryptedContent());
        
        // Update file record
        file.setStorageKey(newStorageKey);
        file.setEncryptedDek(newEncryption.getEncryptedDek());
        fileRepository.save(file);
        
        // Delete old content
        storageService.delete(file.getStorageKey());
        
        // Clear content from memory
        Arrays.fill(content, (byte) 0);
    }
}
```

### Audit Logging

```java
@Aspect
@Component
public class AuditLoggingAspect {
    
    private final AuditLogRepository auditLogRepository;
    private final SecurityContext securityContext;
    
    @Around("@annotation(Audited)")
    public Object auditOperation(ProceedingJoinPoint joinPoint) throws Throwable {
        Audited annotation = getAnnotation(joinPoint);
        
        AuditLog log = AuditLog.builder()
            .userId(securityContext.getCurrentUserId())
            .action(annotation.action())
            .resourceType(annotation.resourceType())
            .resourceId(extractResourceId(joinPoint))
            .ipAddress(securityContext.getClientIp())
            .userAgent(securityContext.getUserAgent())
            .timestamp(Instant.now())
            .build();
        
        try {
            Object result = joinPoint.proceed();
            log.setStatus("SUCCESS");
            log.setDetails(extractDetails(result));
            return result;
        } catch (Exception e) {
            log.setStatus("FAILURE");
            log.setErrorMessage(e.getMessage());
            throw e;
        } finally {
            auditLogRepository.save(log);
        }
    }
}

// Usage
@Service
public class FileService {
    
    @Audited(action = "FILE_DOWNLOAD", resourceType = "FILE")
    public DownloadResponse downloadFile(String fileId) {
        // Implementation
    }
    
    @Audited(action = "FILE_SHARE", resourceType = "FILE")
    public ShareResponse shareFile(String fileId, ShareRequest request) {
        // Implementation
    }
    
    @Audited(action = "FILE_DELETE", resourceType = "FILE")
    public void deleteFile(String fileId) {
        // Implementation
    }
}
```

---

## 5. End-to-End Simulations

### Simulation 1: Peak Hour Traffic Handling

```
Scenario: Monday 9 AM - 3x normal traffic spike

Initial State:
├── Normal traffic: 70K requests/sec
├── Current pods: API(50), Upload(100), Download(100)
└── Database load: 40%

T+0 minutes: Traffic starts increasing
├── Traffic: 100K requests/sec (+43%)
├── Auto-scaler triggered
├── API pods: 50 → 75
└── Upload pods: 100 → 150

T+2 minutes: Peak traffic
├── Traffic: 210K requests/sec (3x normal)
├── API pods: 75 → 100
├── Upload pods: 150 → 300
├── Download pods: 100 → 250
├── Redis hit rate: 90% (increased from 85%)
└── Database load: 65%

T+5 minutes: Stabilization
├── All pods scaled
├── Latency p99: 180ms (within SLA)
├── Error rate: 0.02%
└── No manual intervention needed

T+60 minutes: Traffic normalizes
├── Traffic: 80K requests/sec
├── Scale-down begins (gradual)
├── Pods return to baseline over 30 minutes
└── Cost: ~$500 extra for the hour

Lessons:
├── Pre-warm caches before known peaks
├── Scale-up aggressive, scale-down conservative
└── Monitor database connection pools closely
```

### Simulation 2: Regional Failover

```
Scenario: US-East region becomes unavailable

RTO (Recovery Time Objective): < 5 minutes (automatic failover to US-West)
RPO (Recovery Point Objective): < 1 minute (async replication lag)

T+0: Failure detected
├── Health checks fail for US-East
├── Alert triggered: "Region US-East unavailable"
├── Affected users: 35% of traffic

T+30 seconds: Automatic failover begins
├── DNS TTL: 60 seconds
├── Route 53 health checks fail
├── Traffic routing to US-West begins
└── Some requests fail during transition

T+2 minutes: Failover complete
├── All traffic routed to US-West
├── US-West auto-scales: 2x capacity
├── Cross-region database replica promoted
└── S3 cross-region replication provides data

T+5 minutes: Stabilization
├── Latency increased: +50ms (cross-country)
├── All services operational
├── User impact: 2 minutes of degraded service
└── No data loss

T+2 hours: US-East recovery
├── Region comes back online
├── Database sync from US-West
├── Gradual traffic shift back
└── Full recovery in 30 minutes

Post-incident:
├── RCA: Network issue at cloud provider
├── Action: Improve health check sensitivity
├── Cost: $10K (extra capacity + data transfer)
└── SLA impact: 99.97% for the day (still within 99.9%)
```

### Simulation 3: Storage Tier Migration

```
Scenario: Automatically move cold files to Glacier

Daily Job: Identify cold files
├── Query: Files not accessed in 180+ days
├── Found: 50 TB of files
└── Estimated savings: $800/month

Step 1: Tag files for migration
├── Add metadata: "tier_migration_pending"
├── Create migration batch
└── Estimate: 5 hours to complete

Step 2: Copy to Glacier
├── S3 lifecycle policy triggers
├── Files copied to Glacier tier
├── Progress tracked in database
└── Rate: 10 TB/hour

Step 3: Update metadata
├── Update file records: storage_tier = "glacier"
├── Update access patterns in cache
└── Log migration for audit

Step 4: Handle access to migrated files
├── User requests cold file
├── System detects Glacier tier
├── Initiate restore (1-5 hours)
├── Notify user: "File will be available in X hours"
├── Send notification when ready
└── File temporarily in hot tier (7 days)

Cost Analysis:
├── Before: 200 PB × $0.023 = $4.6M/month
├── After: 100 PB hot + 100 PB cold
├── New cost: $2.3M + $0.4M = $2.7M/month
└── Savings: $1.9M/month (41%)
```

### Simulation 4: High-Load / Contention Scenario - Viral File Sharing

```
Scenario: Popular file shared on social media, millions download simultaneously
Time: Viral content event

┌─────────────────────────────────────────────────────────────┐
│ T+0s: File shared on Twitter, 1M download requests/minute  │
│ - File ID: file_abc123                                      │
│ - Expected: ~100 downloads/minute normal traffic          │
│ - 10,000x traffic spike                                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+0-30min: Download Request Storm                         │
│ - CDN cache: MISS (file not cached, first time viral)      │
│ - All requests forward to origin                          │
│ - Origin: 1M requests/minute                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Load Handling:                                             │
│                                                              │
│ 1. CDN Caching:                                           │
│    - First 100 requests: Origin fetch (S3)                │
│    - CDN caches file (24 hour TTL)                        │
│    - Next 999,900 requests: CDN HIT (< 10ms)            │
│    - Cache hit rate: 99.99% after warm-up                │
│                                                              │
│ 2. Origin Load (for cache misses):                       │
│    - S3: Handles 1M requests/minute (well within limits)  │
│    - File Service: Metadata lookup only (lightweight)      │
│    - Latency: 50ms (S3) + 5ms (metadata) = 55ms          │
│                                                              │
│ 3. Database Load:                                         │
│    - Download tracking: Async (Kafka events)              │
│    - No blocking database writes                          │
│    - Analytics processed offline                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - 99.99% requests served from CDN (< 10ms)               │
│ - Origin handles < 0.01% (cache misses only)            │
│ - P99 latency: 15ms (excellent)                          │
│ - No service degradation                                   │
│ - CDN absorbs entire traffic spike                        │
└─────────────────────────────────────────────────────────────┘
```

### Simulation 5: Edge Case Scenario - Concurrent File Deletion and Download

```
Scenario: User deletes file while someone else is downloading it
Edge Case: Race condition between deletion and download

┌─────────────────────────────────────────────────────────────┐
│ T+0ms: User A initiates file deletion                      │
│ - File ID: file_xyz789                                     │
│ - DELETE /files/file_xyz789                               │
│ - Service marks as deleted in database                     │
│ - Status: ACTIVE → DELETED                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+10ms: User B requests download (concurrent)            │
│ - GET /files/file_xyz789/download                        │
│ - Request arrives before deletion completes                 │
│ - Database check: Status still ACTIVE (transaction not committed)│
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Download Request Processing                       │
│ - Database query: SELECT * FROM files WHERE id='xyz789'  │
│ - Result: Status = DELETED (deletion committed)            │
│ - Service returns: 404 Not Found or 410 Gone              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Edge Case: Large File Download in Progress                │
│                                                              │
│ Problem: User B started download before deletion          │
│ - Download started: T+0ms (before deletion)              │
│ - Deletion requested: T+5s (during download)             │
│ - Download completes: T+30s (after deletion)              │
│                                                              │
│ Question: Should deletion wait for active downloads?      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Solution: Soft Delete with Grace Period                   │
│                                                              │
│ 1. Soft Delete:                                           │
│    - Mark file as DELETED in database                    │
│    - Set deletion timestamp                              │
│    - File remains in S3 (not immediately deleted)        │
│    - Grace period: 30 days (allows active downloads)      │
│                                                              │
│ 2. Active Download Tracking:                              │
│    - Track active downloads per file                      │
│    - Counter: active_downloads (Redis)                    │
│    - Increment on download start                          │
│    - Decrement on download complete/fail                  │
│                                                              │
│ 3. Hard Delete (after grace period):                     │
│    - Background job runs daily                           │
│    - Find files: DELETED + deletion_date > 30 days      │
│    - Check: active_downloads == 0                        │
│    - If true: Delete from S3, remove from database        │
│    - If false: Retry next day                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - Downloads complete successfully (even after deletion)  │
│ - Files cleaned up after grace period                     │
│ - No data loss or interrupted downloads                   │
│ - Storage cleaned up efficiently                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Cost Analysis

### Infrastructure Cost Breakdown

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                      Monthly Cost Breakdown                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Object Storage (500 PB):                                                     │
│  ├── Hot (100 PB × $0.023/GB)           = $2,300,000                        │
│  ├── Warm (200 PB × $0.0125/GB)         = $2,500,000                        │
│  ├── Cold (200 PB × $0.004/GB)          = $800,000                          │
│  └── Subtotal:                          = $5,600,000                         │
│                                                                               │
│  Compute:                                                                     │
│  ├── API Services (200 pods)            = $200,000                          │
│  ├── Upload/Download (1000 pods)        = $800,000                          │
│  ├── Background Workers (320 pods)      = $160,000                          │
│  ├── Sync Services (100 pods)           = $100,000                          │
│  └── Subtotal:                          = $1,260,000                         │
│                                                                               │
│  Databases:                                                                   │
│  ├── PostgreSQL (360 instances)         = $400,000                          │
│  ├── Redis (200 nodes)                  = $200,000                          │
│  ├── Elasticsearch (113 nodes)          = $250,000                          │
│  ├── Kafka (30 brokers)                 = $100,000                          │
│  └── Subtotal:                          = $950,000                           │
│                                                                               │
│  Network:                                                                     │
│  ├── CDN (30 PB egress)                 = $1,500,000                        │
│  ├── Inter-region transfer              = $500,000                          │
│  ├── Load balancers                     = $50,000                           │
│  └── Subtotal:                          = $2,050,000                         │
│                                                                               │
│  Other:                                                                       │
│  ├── KMS (key operations)               = $50,000                           │
│  ├── Monitoring (Datadog)               = $100,000                          │
│  ├── DNS (Route 53)                     = $20,000                           │
│  └── Subtotal:                          = $170,000                           │
│                                                                               │
│  TOTAL:                                 = $10,030,000/month                  │
│                                                                               │
│  Per DAU (50M):                         = $0.20/user/month                   │
│  Per GB stored:                         = $0.02/GB/month                     │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Cost Optimization Strategies

**Current Monthly Cost:** $10,000,000
**Target Monthly Cost:** $5,400,000 (46% reduction)

**Top 3 Cost Drivers:**
1. **Storage (S3):** $5,500,000/month (55%) - Largest single component
2. **CDN Bandwidth:** $2,500,000/month (25%) - High data transfer costs
3. **Compute (EC2):** $1,250,000/month (13%) - Application servers

**Optimization Strategies (Ranked by Impact):**

1. **Storage Tiering (34% savings on storage):**
   - **Current:** All files in S3 Standard
   - **Optimization:** 
     - Move files not accessed > 90 days to S3 Glacier (80% cheaper)
     - Move files not accessed > 30 days to S3 Infrequent Access (50% cheaper)
   - **Savings:** $1,870,000/month (34% of $5.5M storage)
   - **Trade-off:** Slower access to cold files (acceptable for archive)

2. **Deduplication (30% savings on storage):**
   - **Current:** No deduplication
   - **Optimization:** 
     - Block-level deduplication (average dedup ratio: 30%)
     - Reduce storage by 30%
   - **Savings:** $1,650,000/month (30% of $5.5M storage)
   - **Trade-off:** CPU overhead for hashing (acceptable)

3. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand instances for baseline workload
   - **Optimization:** 
     - 3-year Reserved Instances for baseline (80% of compute)
     - On-demand for spikes
   - **Savings:** $500,000/month (40% of $1.25M compute)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

4. **CDN Optimization (15% savings on bandwidth):**
   - **Current:** 60% cache hit ratio
   - **Optimization:** 
     - Increase cache hit ratio to 75%
     - Reduce origin requests by 15%
   - **Savings:** $375,000/month (15% of $2.5M CDN)
   - **Trade-off:** Slight stale content risk (acceptable)

5. **Right-sizing (15% savings on compute):**
   - **Current:** Over-provisioned for safety
   - **Optimization:** 
     - Analyze actual resource usage
     - Downsize over-provisioned instances
   - **Savings:** $187,500/month (15% of $1.25M compute)
   - **Trade-off:** Need careful monitoring to avoid performance issues

6. **Compression (20% savings on storage):**
   - **Current:** Basic compression
   - **Optimization:** 
     - Compress files before storage (reduce size by 20%)
     - Optimize compression algorithms
   - **Savings:** $1,100,000/month (20% of $5.5M storage)
   - **Trade-off:** Slight CPU overhead (negligible)

**Total Potential Savings:** $5,682,500/month (57% reduction)
**Optimized Monthly Cost:** $4,317,500/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| File Upload (5 MB) | $0.000019 | $0.000008 | 58% |
| File Download (5 MB) | $0.000015 | $0.000006 | 60% |
| File Storage (per GB/month) | $0.023 | $0.010 | 57% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Storage Tiering, Deduplication → $3.52M savings
2. **Phase 2 (Month 2):** Reserved Instances, CDN Optimization → $875K savings
3. **Phase 3 (Month 3):** Compression, Right-sizing → $1.29M savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor storage costs (target < $3.63M/month)
- Monitor cache hit rates (target >75%)
- Review and adjust quarterly

### Cost per Operation

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        Cost per Operation                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  File Upload (5 MB average):                                                  │
│  ├── API processing:        $0.000001                                        │
│  ├── Storage write:         $0.000005                                        │
│  ├── Dedup check:           $0.000002                                        │
│  ├── Event publishing:      $0.000001                                        │
│  ├── Search indexing:       $0.000010                                        │
│  └── Total:                 $0.000019 per upload                             │
│                                                                               │
│  File Download (5 MB average):                                                │
│  ├── API processing:        $0.000001                                        │
│  ├── Storage read:          $0.000004                                        │
│  ├── CDN delivery:          $0.000040                                        │
│  └── Total:                 $0.000045 per download                           │
│                                                                               │
│  Sync Check:                                                                  │
│  ├── API processing:        $0.0000005                                       │
│  ├── Cache lookup:          $0.0000001                                       │
│  └── Total:                 $0.0000006 per sync                              │
│                                                                               │
│  Search Query:                                                                │
│  ├── API processing:        $0.000001                                        │
│  ├── ES query:              $0.000010                                        │
│  └── Total:                 $0.000011 per search                             │
│                                                                               │
│  Monthly Storage (per GB):                                                    │
│  ├── Hot tier:              $0.023                                           │
│  ├── Warm tier:             $0.0125                                          │
│  ├── Cold tier:             $0.004                                           │
│  └── Blended average:       $0.011                                           │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Disaster Recovery

### Backup Strategy

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Backup Strategy                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Object Storage (Files):                                                      │
│  ├── Cross-region replication (real-time)                                    │
│  ├── 3 copies within region                                                  │
│  ├── 2 copies in DR region                                                   │
│  └── RPO: 0 (no data loss)                                                   │
│                                                                               │
│  Metadata Database (PostgreSQL):                                              │
│  ├── Streaming replication to DR region                                      │
│  ├── Point-in-time recovery (30 days)                                        │
│  ├── Daily snapshots (90 days retention)                                     │
│  └── RPO: < 1 second                                                         │
│                                                                               │
│  Search Index (Elasticsearch):                                                │
│  ├── Cross-cluster replication                                               │
│  ├── Can rebuild from PostgreSQL if needed                                   │
│  └── RPO: < 5 minutes                                                        │
│                                                                               │
│  Cache (Redis):                                                               │
│  ├── No backup (can be rebuilt)                                              │
│  ├── Warm-up scripts for recovery                                            │
│  └── RPO: N/A (ephemeral)                                                    │
│                                                                               │
│  Recovery Time Objectives:                                                    │
│  ├── Complete failure: < 1 hour                                              │
│  ├── Single service: < 5 minutes                                             │
│  └── Data corruption: < 4 hours                                              │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] S3 cross-region replication lag < 1 minute
- [ ] Database replication lag < 1 minute
- [ ] CDN origin failover configured
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture file upload/download success rate
   - Document active file count
   - Verify S3 replication lag (< 1 minute)
   - Test file operations (100 sample files)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+5 minutes for single service, T+0 to T+60 minutes for complete failure):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-2 min:** Promote secondary region to primary
   - **T+2-3 min:** Update DNS records (Route53 health checks)
   - **T+3-4 min:** Update CDN origin to point to DR region
   - **T+4-4.5 min:** Verify all services healthy in DR region
   - **T+4.5-5 min:** Resume traffic to DR region
   - **Note:** For complete failure, allow up to 60 minutes for full recovery

4. **Post-Failover Validation (T+5 to T+15 minutes):**
   - Verify RTO < 5 minutes (single service) or < 60 minutes (complete failure): ✅ PASS/FAIL
   - Verify RPO < 1 minute: Check S3 replication lag at failure time
   - Test file upload: Upload 1,000 test files, verify >99% success
   - Test file download: Download 1,000 test files, verify >99% success
   - Test file sync: Verify file sync works correctly across regions
   - Monitor metrics: QPS, latency, error rate return to baseline

5. **Data Integrity Verification:**
   - Compare file count: Pre-failover vs post-failover (should match within 1 min)
   - Spot check: Verify 100 random files accessible
   - Check file metadata: Verify metadata matches pre-failover
   - Test edge cases: Large files, concurrent uploads, version history

6. **Failback Procedure (T+15 to T+20 minutes):**
   - Restore primary region services
   - Sync S3 data from DR to primary
   - Verify replication lag < 1 minute
   - Update DNS and CDN to route traffic back to primary
   - Monitor for 5 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 5 minutes (single service) or < 60 minutes (complete failure): Time from failure to service resumption
- ✅ RPO < 1 minute: Maximum data loss (verified via S3 replication lag)
- ✅ File operations work: >99% files accessible correctly
- ✅ No data loss: File count matches pre-failover (within 1 min window)
- ✅ Service resumes within RTO target: All metrics return to baseline
- ✅ File sync functional: File sync works correctly across regions

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: User Uploads File and Shares It

**Step-by-step:**

1. **User Action**: User uploads file via `POST /v1/files/upload` (chunked upload, 10 MB file)
2. **Upload Service**: 
   - Receives chunks, stores in S3: `s3://files/user123/file_abc123/chunk_001, chunk_002, ...`
   - After all chunks: Assembles file, calculates checksum
   - Creates file record: `INSERT INTO files (id, user_id, name, size, s3_key, checksum)`
   - Updates user quota: `UPDATE users SET storage_used = storage_used + 10MB`
3. **Response**: `201 Created` with `{"file_id": "file_abc123", "name": "document.pdf", "size": 10485760}`
4. **User Action**: User shares file via `POST /v1/files/file_abc123/share` with `{"permissions": "read", "expires_in": 3600}`
5. **Share Service**: 
   - Generates share link: `https://files.com/s/xyz789`
   - Creates share record: `INSERT INTO shares (id, file_id, permissions, expires_at)`
   - Stores in Redis: `SET share:xyz789 {file_id, permissions} TTL 1hour`
6. **Response**: `201 Created` with `{"share_link": "https://files.com/s/xyz789"}`
7. **Recipient accesses file** (via share link):
   - Request: `GET /v1/shares/xyz789`
   - Share Service: Validates share, checks expiration
   - Returns file metadata
   - Client requests file: `GET /v1/files/file_abc123/download`
   - File Service: Streams file from S3 via CDN
8. **Result**: File shared and downloaded successfully

**Total latency: Upload ~30s (10MB), Share ~100ms, Download ~2s (CDN)**

### Journey 2: File Sync (Desktop Client)

**Step-by-step:**

1. **Desktop Client**: Polls for changes every 30 seconds via `GET /v1/files/sync?last_sync=timestamp`
2. **Sync Service**: 
   - Queries database: `SELECT * FROM files WHERE user_id = 'user123' AND updated_at > last_sync`
   - Returns list of changed files
3. **Client detects new file**:
   - Downloads file: `GET /v1/files/file_xyz789/download`
   - Saves to local disk
   - Updates local index
4. **User edits file locally**:
   - Client detects change
   - Uploads file: `PUT /v1/files/file_xyz789` (chunked upload)
   - Server updates file in S3
5. **Result**: Files synced across devices

**Total latency: Sync check ~50ms, Download ~2s, Upload ~30s**

### Failure & Recovery Walkthrough

**Scenario: S3 Service Degradation During Upload**

**RTO (Recovery Time Objective):** < 5 minutes (automatic retry with exponential backoff)  
**RPO (Recovery Point Objective):** 0 (uploads queued, no data loss)

**Timeline:**

```
T+0s:    S3 starts returning 503 errors (throttling)
T+0-10s: File uploads fail
T+10s:   Circuit breaker opens (after 5 consecutive failures)
T+10s:   Uploads queued to local buffer (Kafka)
T+15s:   Retry with exponential backoff (1s, 2s, 4s, 8s)
T+30s:   S3 recovers, accepts requests
T+35s:   Circuit breaker closes (after 3 successful requests)
T+40s:   Queued uploads processed from buffer
T+5min:  All queued uploads completed
```

**What degrades:**
- File uploads fail for 10-30 seconds
- Uploads queued in buffer (no data loss)
- File downloads unaffected (reads from S3/CDN)

**What stays up:**
- File downloads (reads from S3/CDN)
- File metadata operations (PostgreSQL)
- Share link generation
- All read operations

**What recovers automatically:**
- Circuit breaker closes after S3 recovery
- Queued uploads processed automatically
- No manual intervention required

**What requires human intervention:**
- Investigate S3 throttling root cause
- Review capacity planning
- Consider S3 request rate increases

**Cascading failure prevention:**
- Circuit breaker prevents retry storms
- Request queuing prevents data loss
- Exponential backoff reduces S3 load
- Timeouts prevent thread exhaustion

### Disaster Recovery Runbook

```
SCENARIO: Primary Region Complete Failure

DETECTION (T+0):
1. Multiple health checks fail
2. PagerDuty alert triggered
3. On-call engineer paged

ASSESSMENT (T+5 minutes):
1. Confirm region-wide outage
2. Check DR region health
3. Notify incident commander
4. Start incident bridge

FAILOVER (T+10 minutes):
1. Verify DR database is current
2. Update DNS to point to DR region
3. Scale up DR region capacity
4. Verify service health

VALIDATION (T+30 minutes):
1. Test critical user flows
2. Verify data consistency
3. Monitor error rates
4. Confirm customer impact

COMMUNICATION (T+45 minutes):
1. Update status page
2. Notify enterprise customers
3. Post to social media if needed

RECOVERY (T+2-24 hours):
1. Monitor primary region recovery
2. Sync any DR region changes back
3. Plan failback procedure
4. Execute failback during low traffic

POST-INCIDENT:
1. Document timeline
2. Root cause analysis
3. Update runbooks
4. Implement improvements
```

