# Collaborative Document Editor - Interview Grilling Q&A

## Trade-off Questions

### Q1: Why OT instead of CRDT?

**Answer:**

```
I chose Operational Transformation (OT) because:

1. Proven at Scale:
   - Google Docs uses OT
   - Proven with billions of documents
   - Well-understood failure modes

2. Lower Memory Overhead:
   - CRDT requires storing tombstones for deletes
   - OT only stores operations
   - At 10B documents, this matters

3. Better for Rich Text:
   - OT handles formatting naturally
   - CRDT rich text is complex (Peritext)
   - OT has mature libraries

4. Simpler Client:
   - OT client is straightforward
   - CRDT requires more client logic
   - Easier to debug

When would I use CRDT?
├── Peer-to-peer editing (no server)
├── Offline-first applications
├── Simpler data structures (counters, sets)
└── When server ordering isn't available

Trade-offs of OT:
├── Requires central server for ordering
├── Transform functions are complex
├── Harder to prove correctness
└── Server is bottleneck for document
```

### Q2: How do you handle a document with 100 concurrent editors?

**Answer:**

```
100 concurrent editors is extreme but handled:

Challenges:
├── 100 WebSocket connections
├── 100 cursors to render
├── High operation rate
├── Broadcast storm

Solutions:

1. Presence Throttling:
   - Batch cursor updates (every 100ms)
   - Only show nearest 10 cursors
   - Fade distant cursors

2. Operation Batching:
   - Combine rapid keystrokes
   - Batch broadcasts
   - Reduce message count by 80%

3. Dedicated Server:
   - Hot documents get dedicated OT server
   - No sharing with other documents
   - More resources available

4. Sectioned Editing:
   - Suggest users edit different sections
   - Visual indicators of crowded areas
   - "Someone is editing here" warnings

5. Graceful Degradation:
   - Disable presence if > 50 editors
   - Reduce update frequency
   - Prioritize content sync

Implementation:
```java
public void handleHotDocument(String docId, int editorCount) {
    if (editorCount > 50) {
        // Reduce presence updates
        presenceThrottleMs = 500;  // From 100ms
        
        // Batch operations more aggressively
        operationBatchSize = 10;  // From 1
        
        // Warn users
        broadcastWarning(docId, "Many editors, some features limited");
    }
    
    if (editorCount > 20) {
        // Promote to dedicated server
        serverPool.assignDedicated(docId);
    }
}
```
```

### Q3: Why WebSocket instead of Server-Sent Events or Long Polling?

**Answer:**

```
WebSocket is chosen because:

1. Bidirectional:
   - Client sends operations
   - Server sends operations
   - Both directions needed

2. Low Latency:
   - Single persistent connection
   - No HTTP overhead per message
   - < 50ms round trip

3. Efficient:
   - No repeated headers
   - Binary protocol possible
   - Lower bandwidth

Comparison:
┌──────────────────────────────────────────────────────────┐
│ Feature        │ WebSocket │ SSE      │ Long Poll       │
├──────────────────────────────────────────────────────────┤
│ Bidirectional  │ Yes       │ No       │ No              │
│ Latency        │ ~50ms     │ ~100ms   │ ~500ms          │
│ Overhead       │ Low       │ Medium   │ High            │
│ Reconnection   │ Manual    │ Auto     │ Auto            │
│ Browser support│ Universal │ Good     │ Universal       │
└──────────────────────────────────────────────────────────┘

Why not SSE?
├── Need bidirectional for operations
├── Would need separate POST for sending
├── More complex client

Why not Long Polling?
├── Too high latency for real-time
├── Too much overhead at scale
├── Poor user experience
```

### Q4: How do you ensure no operations are lost?

**Answer:**

```
Multiple layers ensure operation durability:

1. Client-Side Buffer:
   - Operations buffered until ACK
   - Retry on disconnect
   - Persist to IndexedDB

2. Server-Side Ordering:
   - Operations assigned version numbers
   - Sequential, no gaps
   - Client detects missing versions

3. Write-Ahead Log:
   - Kafka receives operation first
   - Cassandra for queryable storage
   - Either failure recoverable

4. Acknowledgment Protocol:
   - Server ACKs each operation
   - Client retries unACKed
   - Idempotent application

Flow:
┌─────────────────────────────────────────────────────────┐
│ Client          Server           Kafka      Cassandra  │
│    │               │               │            │      │
│    │──operation───>│               │            │      │
│    │               │──write───────>│            │      │
│    │               │               │──replicate │      │
│    │               │               │            │      │
│    │               │──write────────────────────>│      │
│    │               │               │            │      │
│    │<────ACK───────│               │            │      │
│    │               │               │            │      │
│    │ (remove from  │               │            │      │
│    │  local buffer)│               │            │      │
└─────────────────────────────────────────────────────────┘

If server crashes before ACK:
├── Client retries operation
├── Server checks if already applied (idempotent)
├── If applied, just ACK
├── If not, apply and ACK
```

---

## Scaling Questions

### Q5: How would you handle 10x growth?

**Answer:**

```
Current: 200M DAU, 10.5M ops/sec peak
Target: 2B DAU, 105M ops/sec peak

Phase 1: Horizontal Scaling (Month 1-3)
├── WebSocket servers: 600 → 6000
├── OT servers: 300 → 3000
├── Redis nodes: 800 → 8000
├── Cassandra nodes: 200 → 2000
└── Cost: 10x ($15M/month)

Phase 2: Architecture Optimization (Month 4-6)
├── Regional deployment (reduce latency)
├── Document sharding improvements
├── Operation batching optimization
└── Cost reduction to 7x

Phase 3: Efficiency Gains (Month 7-12)
├── Better compression
├── Smarter caching
├── Reserved capacity
└── Target: 5x cost for 10x scale

Key Challenges:
1. Database sharding (more shards)
2. Cross-region consistency
3. Hot document handling
4. Cost management

Cost Projection:
├── Linear: $15M/month
├── Optimized: $8M/month
└── Per user: $0.004/month (down from $0.0077)
```

### Q6: What breaks first under load?

**Answer:**

```
Failure order during traffic spike:

1. Redis (First to fail)
   - Symptom: High latency, timeouts
   - Cause: Memory pressure, CPU saturation
   - Impact: Document state unavailable
   - Fix: Add nodes, increase memory

2. WebSocket Servers
   - Symptom: Connection failures
   - Cause: Connection limit reached
   - Impact: Users can't connect
   - Fix: Add servers, increase limits

3. OT Servers
   - Symptom: Operation latency increases
   - Cause: CPU saturation
   - Impact: Slow collaboration
   - Fix: Add servers, optimize transforms

4. Cassandra
   - Symptom: Write latency increases
   - Cause: Compaction backlog
   - Impact: Operations not persisted quickly
   - Fix: Add nodes, tune compaction

5. Kafka
   - Symptom: Producer backpressure
   - Cause: Partition saturation
   - Impact: Delayed broadcasts
   - Fix: Add partitions, brokers

Mitigation:
├── Auto-scaling on all tiers
├── Circuit breakers
├── Graceful degradation
└── Load shedding for non-critical features
```

### Q7: How do you handle version history at scale?

**Answer:**

```
Challenge: 10B documents × 100 versions = 1 trillion versions

Storage Strategy:

1. Operation-Based History:
   - Store operations, not full snapshots
   - Replay to reconstruct any version
   - Storage: 100 bytes/op vs 100KB/snapshot

2. Periodic Snapshots:
   - Every 100 operations
   - Every 5 minutes of activity
   - Reduces replay time

3. Tiered Storage:
   - Recent (7 days): Cassandra (fast)
   - Medium (30 days): S3 Standard
   - Old (forever): S3 Glacier

4. On-Demand Reconstruction:
   - Don't store all versions
   - Reconstruct when requested
   - Cache recently accessed

Implementation:
```java
public Document getVersion(String docId, long version) {
    // Try cache first
    Document cached = versionCache.get(docId, version);
    if (cached != null) return cached;
    
    // Find nearest snapshot
    Snapshot snapshot = findNearestSnapshot(docId, version);
    
    // Replay operations from snapshot to target
    List<Operation> ops = getOperationsInRange(
        docId, 
        snapshot.getVersion(), 
        version
    );
    
    Document doc = snapshot.getDocument();
    for (Operation op : ops) {
        doc = apply(doc, op);
    }
    
    // Cache for future requests
    versionCache.put(docId, version, doc);
    
    return doc;
}
```

Cost:
├── Operations storage: 30 TB (30 days)
├── Snapshots: 100 TB
├── Total: $5K/month (vs $1M for full versions)
```

---

## Failure Scenarios

### Q8: What happens if the OT server crashes mid-operation?

**Answer:**

```
Scenario: OT server crashes after receiving operation, before ACK

Client Perspective:
├── Sent operation, waiting for ACK
├── Timeout after 5 seconds
├── Retry operation

Server Perspective:
├── Operation may or may not be persisted
├── New server takes over document
├── Must handle duplicate

Recovery Flow:
1. Client retries with same operation ID
2. New OT server receives operation
3. Check Cassandra: "Is operation ID already applied?"
4. If yes: Just send ACK (idempotent)
5. If no: Apply operation normally

Idempotency Key:
```java
public OperationResult processOperation(Operation op) {
    // Check if already processed
    if (operationRepo.exists(op.getDocumentId(), op.getOperationId())) {
        // Already processed, just ACK
        long version = operationRepo.getVersion(op.getOperationId());
        return new OperationResult(version, op, true);
    }
    
    // Process normally
    return processNewOperation(op);
}
```

Data Loss: None
User Impact: 5-10 second delay
```

### Q9: How do you handle split-brain in multi-region?

**Answer:**

```
Split-brain: Regions can't communicate, both accept writes

Prevention:
1. Single Leader per Document:
   - One region owns each document
   - Writes only to owner region
   - Reads from any region

2. Leader Election:
   - Based on document creator's region
   - Failover if region unavailable
   - Consensus via ZooKeeper/etcd

3. Conflict Detection:
   - Version vectors per region
   - Detect divergence on reconnect
   - Merge using OT

If Split-Brain Occurs:
1. Both regions accept operations
2. Each has different version history
3. On reconnect, detect divergence
4. Merge: Transform region B ops against region A
5. Result: Combined document, no data loss

```java
public void mergeAfterPartition(String docId, 
                                 List<Operation> regionAOps,
                                 List<Operation> regionBOps) {
    // Find common ancestor version
    long commonVersion = findCommonAncestor(regionAOps, regionBOps);
    
    // Get ops since common ancestor
    List<Operation> aOps = getOpsSince(regionAOps, commonVersion);
    List<Operation> bOps = getOpsSince(regionBOps, commonVersion);
    
    // Transform B against A
    for (Operation aOp : aOps) {
        bOps = transformAll(aOp, bOps);
    }
    
    // Apply transformed B ops to A's state
    Document merged = applyAll(getState(docId), bOps);
    
    // Broadcast to all clients
    broadcastMerge(docId, merged);
}
```
```

### Q10: What if Redis loses all data?

**Answer:**

```
Redis stores real-time document state (50M active documents)

Impact:
├── Document state lost
├── Must rebuild from Cassandra
├── 5-10 minute recovery per document

Recovery Strategy:

1. Immediate (First 30 seconds):
   - Detect Redis failure
   - Switch to degraded mode
   - Operations queue in memory

2. Short-term (30 sec - 5 min):
   - Rebuild hot documents on-demand
   - User opens document → rebuild
   - Cache rebuilt state

3. Background (5 min - 1 hour):
   - Rebuild all recently active documents
   - Prioritize by activity level
   - Warm up cache proactively

Rebuild Process:
```java
public DocumentState rebuildState(String docId) {
    // Get latest snapshot from Cassandra
    Snapshot snapshot = cassandra.getLatestSnapshot(docId);
    
    // Get operations since snapshot
    List<Operation> ops = cassandra.getOperationsSince(
        docId, snapshot.getVersion()
    );
    
    // Replay operations
    Document doc = snapshot.getDocument();
    for (Operation op : ops) {
        doc = apply(doc, op);
    }
    
    // Store in Redis
    DocumentState state = new DocumentState(doc, snapshot.getVersion() + ops.size());
    redis.setState(docId, state);
    
    return state;
}
```

User Impact:
├── Active editors: 5-10 second delay
├── New document opens: 2-3 second delay
├── No data loss
└── Full recovery: < 1 hour
```

---

## Design Evolution Questions

### Q11: What would you build first with 4 weeks?

**Answer:**

```
MVP (4 weeks, 4 engineers):

Week 1: Core Infrastructure
├── WebSocket server (basic)
├── Document storage (PostgreSQL)
├── User authentication
└── Basic UI shell

Week 2: Editing
├── Simple text editing (no OT yet)
├── Last-write-wins conflict resolution
├── Auto-save
└── Document CRUD

Week 3: Basic Collaboration
├── Simple OT (insert/delete only)
├── Operation broadcast
├── Cursor presence (basic)
└── 2-3 concurrent editors

Week 4: Polish
├── Error handling
├── Reconnection logic
├── Basic formatting
└── Testing

What's NOT included:
├── Rich text formatting
├── Comments
├── Version history
├── Offline support
├── Search
├── Sharing permissions
├── Mobile support

This MVP:
├── Handles 100 concurrent users
├── 1000 documents
├── Basic collaboration works
├── Foundation for growth
```

### Q12: How does the architecture evolve?

**Answer:**

```
Year 1: MVP to Product
├── 10K users, 100K documents
├── Single server OT
├── PostgreSQL for everything
├── Basic collaboration
└── Cost: $5K/month

Year 2: Growth
├── 1M users, 10M documents
├── Redis for real-time state
├── Cassandra for operations
├── Horizontal OT scaling
├── Rich text, comments
└── Cost: $50K/month

Year 3: Scale
├── 50M users, 500M documents
├── Multi-region deployment
├── Elasticsearch for search
├── Advanced OT optimizations
├── Offline support
└── Cost: $500K/month

Year 4: Enterprise
├── 200M users, 5B documents
├── Enterprise features (SSO, audit)
├── Advanced collaboration
├── AI features (suggestions)
└── Cost: $2M/month

Year 5: Platform
├── 1B users, 10B documents
├── API platform
├── Third-party integrations
├── Real-time analytics
└── Cost: $6M/month

Key Architecture Changes:
├── Year 1→2: Add caching, separate ops storage
├── Year 2→3: Multi-region, search
├── Year 3→4: Enterprise, compliance
├── Year 4→5: Platform, ecosystem
```

---

## Level-Specific Expectations

### L4 (Entry-Level)

Should demonstrate:
- Understanding of real-time communication (WebSocket)
- Basic conflict handling concept
- Simple document storage

May struggle with:
- OT algorithm details
- Scaling WebSocket connections
- Consistency guarantees

### L5 (Mid-Level)

Should demonstrate:
- OT or CRDT understanding
- Caching strategy
- Persistence design
- Failure handling

May struggle with:
- Multi-region consistency
- Cost optimization
- 100+ concurrent editors

### L6 (Senior)

Should demonstrate:
- Deep OT/CRDT knowledge
- Multiple architecture options
- Cost analysis
- Evolution over time
- Trade-off articulation

---

## Summary: Key Points

1. **OT is the core**: Operational Transformation enables real-time collaboration
2. **WebSocket for real-time**: Bidirectional, low-latency communication
3. **Redis for state**: Sub-millisecond access to document state
4. **Cassandra for durability**: Write-optimized operations log
5. **Idempotency**: Every operation has unique ID, safe to retry
6. **Graceful degradation**: Presence optional, content sync critical
7. **Version reconstruction**: Store ops, reconstruct versions on-demand
8. **Cost awareness**: $0.0077/user/month at scale

