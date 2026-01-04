# Collaborative Document Editor - Production Deep Dives (Operations)

## 1. Scaling Strategy

### Horizontal Scaling Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Scaling Architecture                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  WebSocket Tier (Connection-bound):                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  • 30M concurrent connections                                        │     │
│  │  • 50K connections/server                                            │     │
│  │  • 600 servers minimum                                               │     │
│  │  • Scale by adding servers                                           │     │
│  │  • Sticky sessions by document_id                                    │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  OT Engine Tier (CPU-bound):                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  • 10.5M operations/second peak                                      │     │
│  │  • 50K ops/second/server                                             │     │
│  │  • 300 servers at peak                                               │     │
│  │  • Scale by document partitioning                                    │     │
│  │  • Stateless transformation                                          │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  Storage Tier (IO-bound):                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  Redis: 800 nodes (real-time state)                                  │     │
│  │  Cassandra: 200 nodes (operations log)                               │     │
│  │  PostgreSQL: 420 instances (metadata)                                │     │
│  │  Elasticsearch: 525 nodes (search)                                   │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Document Partitioning

```java
@Service
public class DocumentPartitioner {
    
    private final int numPartitions = 1000;
    private final ConsistentHash<String> consistentHash;
    
    /**
     * Determine which OT server handles a document
     */
    public String getPartition(String documentId) {
        return consistentHash.get(documentId);
    }
    
    /**
     * Route operation to correct OT server
     */
    public OTServer getServerForDocument(String documentId) {
        String partition = getPartition(documentId);
        return serverRegistry.getServer(partition);
    }
    
    /**
     * Handle hot documents (many concurrent editors)
     */
    public void handleHotDocument(String documentId, int editorCount) {
        if (editorCount > 50) {
            // Promote to dedicated server
            dedicatedServerPool.assign(documentId);
            metrics.increment("hot_document_promoted");
        }
    }
}
```

### Auto-Scaling Configuration

```yaml
# Kubernetes HPA for WebSocket servers
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: websocket-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: websocket-gateway
  minReplicas: 100
  maxReplicas: 800
  metrics:
  - type: Pods
    pods:
      metric:
        name: websocket_connections
      target:
        type: AverageValue
        averageValue: 40000  # Scale at 80% of 50K capacity
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100  # Can double
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

### Circuit Breaker for External Services

```java
@Service
public class ResilientOTService {
    
    private final CircuitBreaker redisCircuitBreaker;
    private final CircuitBreaker cassandraCircuitBreaker;
    
    public ResilientOTService() {
        this.redisCircuitBreaker = CircuitBreaker.of("redis", CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .slidingWindowSize(100)
            .build());
        
        this.cassandraCircuitBreaker = CircuitBreaker.of("cassandra", CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(60))
            .slidingWindowSize(100)
            .build());
    }
    
    public DocumentState getState(String documentId) {
        return redisCircuitBreaker.executeSupplier(() -> {
            DocumentState state = redisCache.getState(documentId);
            if (state != null) {
                return state;
            }
            // Fallback to Cassandra
            return loadFromCassandra(documentId);
        });
    }
    
    @Recover
    public DocumentState getStateFallback(String documentId, Exception e) {
        log.warn("Redis unavailable, loading from Cassandra: {}", documentId);
        return cassandraCircuitBreaker.executeSupplier(() -> 
            loadFromCassandra(documentId)
        );
    }
    
    private DocumentState loadFromCassandra(String documentId) {
        // Get latest snapshot
        VersionSnapshot snapshot = cassandraRepo.getLatestSnapshot(documentId);
        
        // Replay operations since snapshot
        List<Operation> ops = cassandraRepo.getOperationsSince(
            documentId, snapshot.getVersion()
        );
        
        Document doc = snapshot.getDocument();
        for (Operation op : ops) {
            doc = applyOperation(doc, op);
        }
        
        return new DocumentState(documentId, doc, snapshot.getVersion() + ops.size());
    }
}
```

### Graceful Degradation

```java
@Service
public class CollaborationService {
    
    private final FeatureFlags featureFlags;
    
    public EditResponse handleEdit(EditRequest request) {
        EditResponse response = new EditResponse();
        
        // Core functionality - always works
        OperationResult result = otEngine.processOperation(request);
        response.setVersion(result.getVersion());
        response.setAcknowledged(true);
        
        // Presence - degrade gracefully
        if (featureFlags.isEnabled("presence")) {
            try {
                response.setPresence(presenceService.getPresence(request.getDocumentId()));
            } catch (Exception e) {
                log.warn("Presence unavailable, degrading gracefully");
                response.setPresence(Collections.emptyList());
                response.addWarning("Presence temporarily unavailable");
            }
        }
        
        // Comments - degrade gracefully
        if (featureFlags.isEnabled("comments")) {
            try {
                response.setCommentCount(commentService.getCount(request.getDocumentId()));
            } catch (Exception e) {
                log.warn("Comments unavailable");
                response.setCommentCount(null);
            }
        }
        
        // Spell check - optional
        if (featureFlags.isEnabled("spellcheck") && 
            !circuitBreaker("spellcheck").isOpen()) {
            try {
                response.setSpellingSuggestions(
                    spellCheckService.check(request.getContent())
                );
            } catch (Exception e) {
                // Silently degrade
            }
        }
        
        return response;
    }
}
```

### Operation Durability

```java
@Service
public class DurableOperationService {
    
    private final KafkaTemplate<String, Operation> kafka;
    private final CassandraOperationRepository cassandra;
    
    /**
     * Ensure operation is durably stored before acknowledging
     */
    @Transactional
    public void persistOperation(String documentId, long version, Operation op) {
        // Write to Kafka first (fast, durable)
        CompletableFuture<SendResult<String, Operation>> kafkaFuture = 
            kafka.send("document-operations", documentId, op);
        
        // Write to Cassandra (slower, queryable)
        CompletableFuture<Void> cassandraFuture = 
            cassandra.saveAsync(documentId, version, op);
        
        // Wait for at least one to complete
        try {
            CompletableFuture.anyOf(kafkaFuture, cassandraFuture)
                .get(100, TimeUnit.MILLISECONDS);
        } catch (TimeoutException e) {
            // Both slow, but operation is in flight
            log.warn("Slow persistence for doc={}, version={}", documentId, version);
        }
        
        // Background: ensure both complete
        CompletableFuture.allOf(kafkaFuture, cassandraFuture)
            .whenComplete((result, error) -> {
                if (error != null) {
                    log.error("Failed to persist operation", error);
                    alerting.sendAlert("Operation persistence failure");
                }
            });
    }
}
```

---

## 3. Monitoring and Observability

### Key Metrics

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Key Metrics                                         │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Real-time Collaboration:                                                     │
│  ├── operation_latency_ms (histogram)                                        │
│  ├── operations_per_second (counter)                                         │
│  ├── transform_latency_ms (histogram)                                        │
│  ├── broadcast_latency_ms (histogram)                                        │
│  └── conflict_rate (gauge)                                                   │
│                                                                               │
│  WebSocket:                                                                   │
│  ├── active_connections (gauge)                                              │
│  ├── connections_per_document (histogram)                                    │
│  ├── message_rate (counter)                                                  │
│  ├── connection_errors (counter)                                             │
│  └── reconnection_rate (counter)                                             │
│                                                                               │
│  Document State:                                                              │
│  ├── active_documents (gauge)                                                │
│  ├── hot_documents (gauge) - > 10 editors                                   │
│  ├── state_cache_hit_rate (gauge)                                           │
│  ├── snapshot_creation_rate (counter)                                        │
│  └── version_lag (histogram) - client vs server                             │
│                                                                               │
│  Persistence:                                                                 │
│  ├── operation_persist_latency_ms (histogram)                               │
│  ├── cassandra_write_latency_ms (histogram)                                 │
│  ├── redis_latency_ms (histogram)                                           │
│  └── kafka_produce_latency_ms (histogram)                                   │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Alerting Rules

```yaml
groups:
- name: collaborative-editor-alerts
  rules:
  
  # OT latency
  - alert: HighOperationLatency
    expr: |
      histogram_quantile(0.99, rate(operation_latency_ms_bucket[5m])) > 500
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "P99 operation latency > 500ms"
      
  # WebSocket connections
  - alert: WebSocketConnectionSpike
    expr: |
      rate(active_connections[5m]) > 100000
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Rapid WebSocket connection increase"
      
  # Document state
  - alert: HighVersionLag
    expr: |
      histogram_quantile(0.95, version_lag) > 10
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Clients falling behind server version"
      
  # Redis health
  - alert: RedisCacheHighLatency
    expr: |
      histogram_quantile(0.99, redis_latency_ms_bucket[5m]) > 10
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Redis P99 latency > 10ms"
      
  # Cassandra health
  - alert: CassandraWriteLatency
    expr: |
      histogram_quantile(0.99, cassandra_write_latency_ms_bucket[5m]) > 100
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Cassandra write P99 > 100ms"
```

### Distributed Tracing

```java
@Service
public class TracedOTEngine {
    
    private final Tracer tracer;
    
    public OperationResult processOperation(OperationRequest request) {
        Span span = tracer.spanBuilder("process-operation")
            .setAttribute("document.id", request.getDocumentId())
            .setAttribute("client.version", request.getClientVersion())
            .setAttribute("operation.type", request.getOperation().getType())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // Get state
            Span stateSpan = tracer.spanBuilder("get-state").startSpan();
            DocumentState state = stateCache.getState(request.getDocumentId());
            stateSpan.setAttribute("cache.hit", state != null);
            stateSpan.end();
            
            // Transform
            Span transformSpan = tracer.spanBuilder("transform-operation").startSpan();
            Operation transformed = transform(state, request);
            transformSpan.setAttribute("ops.transformed", getTransformCount());
            transformSpan.end();
            
            // Persist
            Span persistSpan = tracer.spanBuilder("persist-operation").startSpan();
            persistOperation(request.getDocumentId(), state.getVersion() + 1, transformed);
            persistSpan.end();
            
            // Broadcast
            Span broadcastSpan = tracer.spanBuilder("broadcast-operation").startSpan();
            broadcastOperation(request.getDocumentId(), transformed);
            broadcastSpan.setAttribute("clients.count", getClientCount(request.getDocumentId()));
            broadcastSpan.end();
            
            span.setAttribute("result.version", state.getVersion() + 1);
            return new OperationResult(state.getVersion() + 1, transformed);
            
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

---

## 4. Security Deep Dive

### Document Access Control

```java
@Service
public class DocumentAccessControl {
    
    public enum Permission {
        VIEW,      // Read document
        COMMENT,   // Add comments
        EDIT,      // Modify content
        SHARE,     // Share with others
        ADMIN      // Manage permissions
    }
    
    public boolean checkAccess(String userId, String documentId, Permission required) {
        // Get document
        Document doc = documentRepository.findById(documentId)
            .orElseThrow(() -> new DocumentNotFoundException(documentId));
        
        // Owner has all permissions
        if (doc.getOwnerId().equals(userId)) {
            return true;
        }
        
        // Check direct share
        Optional<DocumentShare> share = shareRepository
            .findByDocumentAndUser(documentId, userId);
        
        if (share.isPresent()) {
            return hasPermission(share.get().getRole(), required);
        }
        
        // Check link access (for anonymous users)
        String shareToken = RequestContext.getShareToken();
        if (shareToken != null) {
            ShareLink link = shareLinkRepository.findByToken(shareToken);
            if (link != null && link.getDocumentId().equals(documentId)) {
                return hasPermission(link.getRole(), required);
            }
        }
        
        return false;
    }
    
    private boolean hasPermission(String role, Permission required) {
        Set<Permission> rolePermissions = switch (role) {
            case "owner" -> EnumSet.allOf(Permission.class);
            case "editor" -> EnumSet.of(VIEW, COMMENT, EDIT, SHARE);
            case "commenter" -> EnumSet.of(VIEW, COMMENT);
            case "viewer" -> EnumSet.of(VIEW);
            default -> EnumSet.noneOf(Permission.class);
        };
        return rolePermissions.contains(required);
    }
}
```

### Operation Validation

```java
@Service
public class OperationValidator {
    
    public void validateOperation(String userId, String documentId, Operation op) {
        // Check edit permission
        if (!accessControl.checkAccess(userId, documentId, Permission.EDIT)) {
            throw new AccessDeniedException("No edit permission");
        }
        
        // Validate operation structure
        validateOperationStructure(op);
        
        // Check content policies
        if (op instanceof InsertOperation) {
            InsertOperation insert = (InsertOperation) op;
            validateContent(insert.getContent());
        }
        
        // Rate limiting
        if (!rateLimiter.tryAcquire(userId, documentId)) {
            throw new RateLimitedException("Too many operations");
        }
        
        // Size limits
        Document doc = getDocument(documentId);
        if (doc.getSize() + op.getSizeImpact() > MAX_DOCUMENT_SIZE) {
            throw new DocumentTooLargeException("Document would exceed size limit");
        }
    }
    
    private void validateContent(String content) {
        // Check for malicious content
        if (containsScriptTags(content)) {
            throw new InvalidContentException("Script tags not allowed");
        }
        
        // Check for PII (if enabled)
        if (piiDetector.containsPII(content)) {
            auditLog.logPIIDetected(content);
        }
    }
}
```

## 4.5. Simulation (End-to-End User Journeys)

### Journey 1: User Edits Document (Happy Path)

**Step-by-step:**

1. **User Action**: Opens document, types text
2. **Client**: Captures edit operation (InsertOperation)
3. **WebSocket Service**: 
   - Sends operation to server
   - Server validates and applies operation
   - Broadcasts to other collaborators
4. **Other Users**: 
   - Receive operation via WebSocket
   - Apply operation transformation (OT)
   - Update local document state
5. **Persistence**: 
   - Operation stored in database
   - Document state updated
6. **Response**: All users see changes in real-time (< 100ms)

### Journey 2: Concurrent Editing with Conflict Resolution

**Step-by-step:**

1. **User A**: Inserts "Hello" at position 5
2. **User B**: Inserts "World" at position 5 (concurrent, same position)
3. **OT Service**: 
   - Transforms operations
   - User A's operation: Insert("Hello", 5)
   - User B's operation: Insert("World", 5) → Transform → Insert("World", 10)
4. **Result**: Document shows "HelloWorld" (both edits preserved)

### High-Load / Contention Scenario: Large Team Concurrent Editing

```
Scenario: 100 users edit document simultaneously during meeting
Time: Live collaboration session (peak traffic)

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Meeting starts, 100 users join document              │
│ - Document ID: doc_xyz789                                  │
│ - Users: 100 concurrent editors                            │
│ - Expected: ~10 concurrent users normal traffic            │
│ - 10x traffic spike                                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+0-60min: Operation Storm                                │
│ - 100 users typing simultaneously                         │
│ - Operation rate: 5,000 operations/minute                 │
│ - WebSocket messages: 5,000 messages/minute               │
│ - Each operation: Validate → Transform → Broadcast        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Load Handling:                                             │
│                                                              │
│ 1. Operation Batching:                                    │
│    - Client batches operations (50ms window)              │
│    - Sends batch of 5-10 operations at once              │
│    - Reduces WebSocket message count by 80%              │
│    - Server processes batches efficiently                  │
│                                                              │
│ 2. OT Transformation:                                     │
│    - Operations transformed per user                      │
│    - OT algorithm: O(log n) complexity                   │
│    - 100 users: 100 transformations per operation         │
│    - CPU load: 5000 ops/min × 100 transforms = 500K/min │
│    - Server: 20 cores × 50K transforms/sec = 1M/min     │
│    - Utilization: 50% (safe)                             │
│                                                              │
│ 3. WebSocket Broadcast:                                  │
│    - 5,000 operations/minute × 100 users = 500K msgs/min│
│    - WebSocket servers: 10 instances                     │
│    - Each server: 50K messages/minute                    │
│    - Each server: 833 messages/second                   │
│    - Capacity: 5,000 messages/second per server (safe)   │
│                                                              │
│ 4. Database Writes:                                      │
│    - Operations stored in database (async)                │
│    - Batch inserts: 100 operations per batch             │
│    - Write rate: 50 batches/minute                      │
│    - Database: Handles easily (low write rate)           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - All 100 users edit smoothly                            │
│ - Operations applied in correct order                    │
│ - P99 latency: 150ms (under 200ms target)               │
│ - No conflicts or data loss                              │
│ - System handles 10x normal load gracefully             │
└─────────────────────────────────────────────────────────────┘
```

### Edge Case Scenario: Operation Transformation Conflict

```
Scenario: Two users perform conflicting operations that OT cannot resolve cleanly
Edge Case: Complex operation transformation results in unexpected document state

┌─────────────────────────────────────────────────────────────┐
│ T+0ms: User A selects text "Hello World"                  │
│ - Selection: Positions 0-11                               │
│ - Operation prepared: Delete(0, 11)                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+50ms: User B inserts "Hi " at position 0 (concurrent) │
│ - Operation: Insert("Hi ", 0)                            │
│ - Document state: "Hi Hello World"                      │
│ - User A's selection now invalid (positions shifted)      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+100ms: User A's delete operation arrives              │
│ - Operation: Delete(0, 11)                               │
│ - Current document: "Hi Hello World"                    │
│ - Problem: Delete(0, 11) now deletes "Hi Hello W"      │
│ - User A intended to delete "Hello World" (original)    │
│ - Result: Wrong text deleted                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Edge Case: Selection-Based Operation Conflict            │
│                                                              │
│ Problem: Selection-based operations don't transform well  │
│ - Delete(0, 11) based on selection                       │
│ - After transform: Delete(3, 14) (wrong range)          │
│ - User intent lost in transformation                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Solution: Operational Transformation with Undo/Redo      │
│                                                              │
│ 1. Operation Context:                                    │
│    - Include operation context (selected text)           │
│    - Store: Delete(0, 11, context="Hello World")        │
│    - Transform using context, not just positions         │
│                                                              │
│ 2. Conflict Resolution:                                  │
│    - If transform results in invalid operation:          │
│      a. Try to match by content (string matching)        │
│      b. If content changed: Mark operation as conflicted │
│      c. Notify user: "Operation conflicted, please retry"│
│                                                              │
│ 3. Undo/Redo Support:                                   │
│    - Store operation history with undo stack            │
│    - If conflict detected: Allow undo                    │
│    - User can undo and redo operation                    │
│    - Better UX than silent failure                       │
│                                                              │
│ 4. Content-Aware Transformation:                        │
│    - Use CRDT (Conflict-free Replicated Data Type)      │
│    - Operations based on content, not positions         │
│    - Delete("Hello World") instead of Delete(0, 11)    │
│    - More robust conflict resolution                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - Conflicts detected and handled gracefully             │
│ - Users notified of conflicts (transparent)            │
│ - Undo/redo allows recovery from conflicts             │
│ - CRDT approach eliminates most conflicts              │
│ - No silent data corruption                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Cost Analysis

### Infrastructure Cost Breakdown

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                      Monthly Cost Breakdown                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Compute:                                                                     │
│  ├── WebSocket servers (800 × $200)      = $160,000                         │
│  ├── OT Engine servers (300 × $300)      = $90,000                          │
│  ├── API servers (156 × $200)            = $31,200                          │
│  ├── Background workers (320 × $100)     = $32,000                          │
│  └── Subtotal:                           = $313,200                          │
│                                                                               │
│  Databases:                                                                   │
│  ├── PostgreSQL (420 × $500)             = $210,000                         │
│  ├── Cassandra (200 × $400)              = $80,000                          │
│  ├── Redis (800 × $150)                  = $120,000                         │
│  ├── Elasticsearch (525 × $400)          = $210,000                         │
│  └── Subtotal:                           = $620,000                          │
│                                                                               │
│  Messaging:                                                                   │
│  ├── Kafka (100 × $500)                  = $50,000                          │
│  └── Subtotal:                           = $50,000                           │
│                                                                               │
│  Storage:                                                                     │
│  ├── Document storage (5 PB × $20/TB)    = $100,000                         │
│  └── Subtotal:                           = $100,000                          │
│                                                                               │
│  Network:                                                                     │
│  ├── CDN/Load balancing                  = $200,000                         │
│  ├── Data transfer                       = $150,000                         │
│  └── Subtotal:                           = $350,000                          │
│                                                                               │
│  Other:                                                                       │
│  ├── Monitoring (Datadog)                = $100,000                         │
│  ├── DNS, certificates                   = $10,000                          │
│  └── Subtotal:                           = $110,000                          │
│                                                                               │
│  TOTAL:                                  = $1,543,200/month                  │
│                                                                               │
│  Per DAU (200M):                         = $0.0077/user/month                │
│  Per operation (300B/month):             = $0.000005/operation               │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Cost Optimization Strategies

**Current Monthly Cost:** $500,000
**Target Monthly Cost:** $300,000 (40% reduction)

**Top 3 Cost Drivers:**
1. **WebSocket Servers (EC2):** $200,000/month (40%) - Largest single component
2. **Storage (Redis + Cassandra + S3):** $150,000/month (30%) - Document storage
3. **Network (Bandwidth):** $75,000/month (15%) - High data transfer costs

**Optimization Strategies (Ranked by Impact):**

1. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand instances for baseline workload
   - **Optimization:** 
     - 3-year Reserved Instances for baseline (80% of compute)
     - On-demand for peaks
   - **Savings:** $80,000/month (40% of $200K WebSocket servers)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

2. **Tiered Storage (40% savings on storage):**
   - **Current:** All documents in same storage tier
   - **Optimization:** 
     - Hot: Active documents in Redis (10% of documents)
     - Warm: Recent documents in Cassandra (30% of documents)
     - Cold: Old documents in S3 (60% of documents)
   - **Savings:** $60,000/month (40% of $150K storage)
   - **Trade-off:** Cold document load time (acceptable for archive)

3. **Operation Batching (30% savings on bandwidth):**
   - **Current:** Individual operations sent separately
   - **Optimization:** 
     - Batch small operations (typing)
     - Reduce message overhead
   - **Savings:** $22,500/month (30% of $75K network)
   - **Trade-off:** Slightly higher latency (acceptable, < 50ms)

4. **WebSocket Connection Pooling (20% savings on servers):**
   - **Current:** One connection per document
   - **Optimization:** 
     - Multiplex multiple documents per connection
     - Reduce connection overhead
   - **Savings:** $40,000/month (20% of $200K WebSocket servers)
   - **Trade-off:** More complex client (acceptable)

5. **Regional Optimization (20% savings on network):**
   - **Current:** Single region deployment
   - **Optimization:** 
     - Deploy closer to users
     - Reduce latency and bandwidth
   - **Savings:** $15,000/month (20% of $75K network)
   - **Trade-off:** Operational complexity (acceptable)

6. **Right-sizing:**
   - **Current:** Over-provisioned for safety
   - **Optimization:** 
     - Monitor actual usage for 1 month
     - Reduce instance sizes where possible
   - **Savings:** $20,000/month (10% of $200K compute)
   - **Trade-off:** Need careful monitoring to avoid performance issues

**Total Potential Savings:** $237,500/month (48% reduction)
**Optimized Monthly Cost:** $262,500/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Edit Operation | $0.000001 | $0.0000005 | 50% |
| Document Open | $0.00001 | $0.000005 | 50% |
| WebSocket Connection (per hour) | $0.0001 | $0.00006 | 40% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Reserved Instances, Tiered Storage → $140K savings
2. **Phase 2 (Month 2):** Operation Batching, WebSocket Pooling → $62.5K savings
3. **Phase 3 (Month 3):** Regional Optimization, Right-sizing → $35K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor WebSocket connection count (target < 20M concurrent)
- Monitor storage costs (target < $90K/month)
- Review and adjust quarterly

---

## 6. Disaster Recovery

### Backup Strategy

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Backup Strategy                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Document Content:                                                            │
│  ├── Continuous: Operations to Kafka (replicated)                            │
│  ├── Hourly: Snapshots to S3                                                 │
│  ├── Daily: Full backup to Glacier                                           │
│  └── RPO: 0 (no data loss with Kafka)                                        │
│                                                                               │
│  Operations Log (Cassandra):                                                  │
│  ├── Replication factor: 3                                                   │
│  ├── Cross-DC replication                                                    │
│  ├── Incremental backups daily                                               │
│  └── RPO: < 1 second                                                         │
│                                                                               │
│  Metadata (PostgreSQL):                                                       │
│  ├── Streaming replication to standby                                        │
│  ├── Point-in-time recovery (30 days)                                        │
│  ├── Daily snapshots                                                         │
│  └── RPO: < 1 second                                                         │
│                                                                               │
│  Real-time State (Redis):                                                     │
│  ├── No backup (reconstructible)                                             │
│  ├── Rebuild from Cassandra operations                                       │
│  └── Recovery time: < 5 minutes per document                                 │
│                                                                               │
│  Recovery Time Objectives:                                                    │
│  ├── Single service failure: < 1 minute (auto-failover)                     │
│  ├── Zone failure: < 5 minutes                                               │
│  ├── Region failure: < 30 minutes                                            │
│  └── Complete rebuild: < 4 hours                                             │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Kafka replication lag < 1 minute
- [ ] Database replication lag < 1 minute
- [ ] Redis replication lag < 1 minute
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available
- [ ] Minimal active editing sessions during test window

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture document editing success rate
   - Document active document count
   - Verify Kafka replication lag (< 1 minute)
   - Test document editing (100 sample documents)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+5 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-2 min:** Promote secondary region to primary
   - **T+2-3 min:** Update DNS records (Route53 health checks)
   - **T+3-4 min:** Rebalance Kafka partitions
   - **T+4-4.5 min:** Warm up Redis cache from database
   - **T+4.5-5 min:** Verify all services healthy in DR region
   - **T+5 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+5 to T+15 minutes):**
   - Verify RTO < 5 minutes: ✅ PASS/FAIL
   - Verify RPO < 1 minute: Check Kafka replication lag at failure time
   - Test document editing: Edit 1,000 test documents, verify >99% success
   - Test operational transform: Verify OT works correctly post-failover
   - Test conflict resolution: Verify conflict resolution works correctly
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify real-time collaboration: Check WebSocket connections functional

5. **Data Integrity Verification:**
   - Compare document count: Pre-failover vs post-failover (should match within 1 min)
   - Spot check: Verify 100 random documents accessible
   - Check document state: Verify document state matches pre-failover
   - Test edge cases: Concurrent edits, large documents, version history

6. **Failback Procedure (T+15 to T+20 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag < 1 minute
   - Update DNS to route traffic back to primary
   - Monitor for 5 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 5 minutes: Time from failure to service resumption
- ✅ RPO < 1 minute: Maximum data loss (verified via Kafka replication lag)
- ✅ Document editing works: >99% edits successful
- ✅ No data loss: Document count matches pre-failover (within 1 min window)
- ✅ Service resumes within RTO target: All metrics return to baseline
- ✅ Real-time collaboration functional: WebSocket connections work correctly

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

### Failover Procedure

**RTO (Recovery Time Objective):** < 5 minutes (manual failover to DR region)  
**RPO (Recovery Point Objective):** < 1 minute (async replication lag)

```
SCENARIO: Primary region failure

DETECTION (T+0):
1. Health checks fail across multiple services
2. PagerDuty alert triggered
3. On-call engineer paged

ASSESSMENT (T+2 minutes):
1. Confirm region-wide outage
2. Check DR region health
3. Initiate incident response

FAILOVER (T+5 minutes):
1. Update DNS to DR region
2. Promote Cassandra DR cluster
3. Promote PostgreSQL standby

### Database Connection Pool Configuration

**Connection Pool Settings (HikariCP):**

```yaml
spring:
  datasource:
    hikari:
      # Pool sizing
      maximum-pool-size: 20          # Max connections per application instance
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

For our service (420 PostgreSQL instances):
- 4 CPU cores per pod
- Spindle count = 1 per instance
- Calculated: (4 × 2) + 1 = 9 connections minimum
- With safety margin: 20 connections per pod
- 300 pods × 20 = 6000 max connections distributed across 420 instances
- Per-instance average: ~14 connections per instance
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
   - Place PgBouncer between app and PostgreSQL instances
   - PgBouncer pool: 30 connections per instance
   - App pools can share PgBouncer connections efficiently
4. Scale up DR compute

VALIDATION (T+15 minutes):
1. Verify document access
2. Test real-time collaboration
3. Check operation persistence
4. Monitor error rates

USER IMPACT:
├── Active sessions: Disconnected, auto-reconnect
├── Unsent operations: Buffered on client, retried
├── Data loss: None (Kafka replication)
└── Total downtime: ~15 minutes
```

