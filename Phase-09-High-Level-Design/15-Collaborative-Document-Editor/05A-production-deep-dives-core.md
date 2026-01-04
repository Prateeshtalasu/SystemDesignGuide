# Collaborative Document Editor - Production Deep Dives (Core)

## 1. Operational Transformation Deep Dive

### OT Fundamentals

```
What is Operational Transformation?

OT is a technique for enabling real-time collaborative editing.
It transforms operations to account for concurrent changes,
ensuring all clients converge to the same document state.

Key Properties:
1. Convergence: All clients reach the same state
2. Intention Preservation: User's intent is maintained
3. Causality: Cause always precedes effect

The Transform Function:
- Input: Two concurrent operations (A, B)
- Output: Transformed operations (A', B')
- Property: apply(apply(doc, A), B') = apply(apply(doc, B), A')
```

### OT Implementation

```java
public class OTEngine {
    
    // Document state per document
    private final ConcurrentHashMap<String, DocumentState> documents;
    private final OperationRepository operationRepo;
    private final RedisTemplate<String, Object> redis;
    
    /**
     * Core OT processing loop
     */
    public synchronized OperationResult processOperation(
            String documentId,
            String clientId,
            long clientVersion,
            Operation clientOp) {
        
        DocumentState state = getOrLoadState(documentId);
        
        // Step 1: Get server version
        long serverVersion = state.getVersion();
        
        // Step 2: Get operations client hasn't seen
        List<Operation> serverOps = operationRepo
            .getOperationsInRange(documentId, clientVersion + 1, serverVersion);
        
        // Step 3: Transform client operation against server operations
        Operation transformed = clientOp;
        for (Operation serverOp : serverOps) {
            transformed = transform(serverOp, transformed);
        }
        
        // Step 4: Apply transformed operation
        Document newDoc = applyOperation(state.getDocument(), transformed);
        
        // Step 5: Update state
        long newVersion = serverVersion + 1;
        state.setDocument(newDoc);
        state.setVersion(newVersion);
        
        // Step 6: Persist operation
        operationRepo.save(new PersistedOperation(
            documentId,
            newVersion,
            transformed,
            clientId,
            Instant.now()
        ));
        
        // Step 7: Broadcast to other clients
        broadcastOperation(documentId, clientId, newVersion, transformed);
        
        return new OperationResult(newVersion, transformed);
    }
    
    /**
     * Transform operation B against operation A
     * Returns B' such that the OT property holds
     */
    private Operation transform(Operation a, Operation b) {
        // Handle all operation type combinations
        if (a instanceof Insert && b instanceof Insert) {
            return transformII((Insert) a, (Insert) b);
        } else if (a instanceof Insert && b instanceof Delete) {
            return transformID((Insert) a, (Delete) b);
        } else if (a instanceof Delete && b instanceof Insert) {
            return transformDI((Delete) a, (Insert) b);
        } else if (a instanceof Delete && b instanceof Delete) {
            return transformDD((Delete) a, (Delete) b);
        }
        throw new IllegalStateException("Unknown operation types");
    }
    
    /**
     * Transform Insert against Insert
     */
    private Insert transformII(Insert a, Insert b) {
        if (a.getPosition() < b.getPosition()) {
            // A is before B, shift B right
            return new Insert(
                b.getPosition() + a.getText().length(),
                b.getText(),
                b.getAttributes()
            );
        } else if (a.getPosition() > b.getPosition()) {
            // A is after B, no change
            return b;
        } else {
            // Same position - use client ID for deterministic ordering
            if (a.getClientId().compareTo(b.getClientId()) < 0) {
                return new Insert(
                    b.getPosition() + a.getText().length(),
                    b.getText(),
                    b.getAttributes()
                );
            }
            return b;
        }
    }
    
    /**
     * Transform Delete against Insert
     */
    private Delete transformDI(Delete a, Insert b) {
        if (b.getPosition() <= a.getPosition()) {
            // Insert before delete, shift delete right
            return new Delete(
                a.getPosition() + b.getText().length(),
                a.getLength()
            );
        } else if (b.getPosition() >= a.getPosition() + a.getLength()) {
            // Insert after delete, no change
            return a;
        } else {
            // Insert within delete range - split delete
            int beforeLength = b.getPosition() - a.getPosition();
            int afterLength = a.getLength() - beforeLength;
            // Return compound operation
            return new CompoundDelete(
                new Delete(a.getPosition(), beforeLength),
                new Delete(
                    a.getPosition() + beforeLength + b.getText().length(),
                    afterLength
                )
            );
        }
    }
}
```

### Client-Side OT

```javascript
class OTClient {
    constructor(documentId, websocket) {
        this.documentId = documentId;
        this.ws = websocket;
        this.document = null;
        this.version = 0;
        this.pendingOps = [];      // Sent but not acknowledged
        this.buffer = [];          // Not yet sent
        this.inflightOp = null;    // Currently being sent
    }
    
    /**
     * User makes a local edit
     */
    applyLocal(operation) {
        // Apply immediately to local document
        this.document = this.apply(this.document, operation);
        
        // Buffer the operation
        this.buffer.push(operation);
        
        // Try to send
        this.flushBuffer();
    }
    
    /**
     * Send buffered operations to server
     */
    flushBuffer() {
        if (this.inflightOp !== null || this.buffer.length === 0) {
            return; // Wait for ACK or nothing to send
        }
        
        // Compose all buffered operations
        this.inflightOp = this.compose(this.buffer);
        this.buffer = [];
        
        // Send to server
        this.ws.send({
            type: 'operation',
            version: this.version,
            operation: this.inflightOp
        });
    }
    
    /**
     * Receive operation from server
     */
    receiveOperation(serverVersion, serverOp, isAck) {
        if (isAck) {
            // Our operation was acknowledged
            this.version = serverVersion;
            this.inflightOp = null;
            this.flushBuffer();
            return;
        }
        
        // Server operation from another client
        this.version = serverVersion;
        
        // Transform against our pending operations
        let transformedServerOp = serverOp;
        
        if (this.inflightOp) {
            const [newInflight, newServer] = this.transformPair(
                this.inflightOp, 
                transformedServerOp
            );
            this.inflightOp = newInflight;
            transformedServerOp = newServer;
        }
        
        // Transform against buffer
        const newBuffer = [];
        for (const bufOp of this.buffer) {
            const [newBuf, newServer] = this.transformPair(
                bufOp, 
                transformedServerOp
            );
            newBuffer.push(newBuf);
            transformedServerOp = newServer;
        }
        this.buffer = newBuffer;
        
        // Apply transformed server operation
        this.document = this.apply(this.document, transformedServerOp);
        
        // Update UI
        this.render();
    }
}
```

---

## 1.5. Operational Transformation - Technology-Level Simulation

### C) REAL STEP-BY-STEP SIMULATION: OT Technology-Level

**Normal Flow: Concurrent Edits with OT**

```
Step 1: Two Users Edit Simultaneously
┌─────────────────────────────────────────────────────────────┐
│ Document: "Hello" (version 10)                              │
│ Alice types "A" at position 0                               │
│ Bob types "B" at position 0                                 │
│ Both operations sent to server                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Server Receives Alice's Operation First
┌─────────────────────────────────────────────────────────────┐
│ Server:                                                      │
│ - Current version: 10                                       │
│ - Client version: 10 (matches)                              │
│ - No transformation needed (first operation)               │
│ - Apply: "Hello" → "AHello"                                │
│ - New version: 11                                           │
│ - Persist to Cassandra                                      │
│ - Broadcast to other clients (Bob)                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Server Receives Bob's Operation
┌─────────────────────────────────────────────────────────────┐
│ Server:                                                      │
│ - Current version: 11 (Alice's op applied)                  │
│ - Client version: 10 (Bob hasn't seen Alice's op)          │
│ - Get missing operations: [insert("A", 0)]                 │
│ - Transform Bob's op against Alice's:                      │
│   Original: insert("B", 0)                                 │
│   Against: insert("A", 0)                                  │
│   Transformed: insert("B", 1)  (shifted right)            │
│ - Apply: "AHello" → "ABHello"                             │
│ - New version: 12                                          │
│ - Persist to Cassandra                                     │
│ - Broadcast to all clients                                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Clients Receive Transformed Operations
┌─────────────────────────────────────────────────────────────┐
│ Alice receives: insert("B", 1), version=12                 │
│ - Current state: "AHello"                                   │
│ - Apply: "AHello" → "ABHello"                              │
│                                                              │
│ Bob receives: insert("A", 0), version=11                    │
│ - Current state: "BHello"                                   │
│ - Apply: "BHello" → "ABHello"                              │
│                                                              │
│ Result: Both clients converge to "ABHello"                  │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Network Partition During Edit**

```
Scenario: Network partition splits users, then reconnects

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Document "Hello" (version 10)                         │
│ T+1s: Network partition                                     │
│   - Alice on partition A                                   │
│   - Bob on partition B                                     │
│ T+2s: Alice types "A" → Applied locally                   │
│ T+3s: Bob types "B" → Applied locally                     │
│ T+4s: Network reconnects                                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+5s: Server receives both operations                      │
│   - Alice: insert("A", 0), version=10                      │
│   - Bob: insert("B", 0), version=10                       │
│ T+6s: Server processes Alice first                         │
│   - Apply: "Hello" → "AHello", version=11                   │
│ T+7s: Server processes Bob                                 │
│   - Transform: insert("B", 0) against insert("A", 0)      │
│   - Result: insert("B", 1)                                 │
│   - Apply: "AHello" → "ABHello", version=12                │
│ T+8s: Both clients sync                                    │
│   - Alice: Receives insert("B", 1) → "ABHello"           │
│   - Bob: Receives insert("A", 0) → "ABHello"              │
│ Result: Converged to "ABHello" (no data loss)              │
└─────────────────────────────────────────────────────────────┘
```

**Idempotency Handling:**

```
Problem: Same operation processed twice (retry)

Solution: Version-based idempotency
1. Each operation has version number
2. Server checks if version already processed
3. Duplicate operations (same version) ignored
4. Operations are idempotent (same version → same result)

Example:
- Operation version 11 processed twice → Second ignored → Idempotent
```

**Traffic Spike Handling:**

```
Scenario: 100 users edit document simultaneously

┌─────────────────────────────────────────────────────────────┐
│ Normal: 10 operations/second                                │
│ Spike: 1,000 operations/second (large team meeting)        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ OT Engine Handling:                                          │
│ - Operations queued per document                            │
│ - Processed sequentially (maintains order)                 │
│ - Transform each operation against all previous             │
│ - Broadcast to all clients                                  │
│                                                              │
│ Bottleneck: Sequential processing                            │
│ Solution: Batch operations (process 10 ops at once)         │
│ Result: 1,000 ops/second handled                            │
└─────────────────────────────────────────────────────────────┘
```

**Hot Document Mitigation:**

```
Problem: Popular document edited by many users simultaneously

Mitigation:
1. Document sharding: Split large documents into sections
2. Operational batching: Batch operations per document
3. Redis pub/sub: Efficient broadcast to all clients

If hot document occurs:
- Monitor operation queue length
- Scale OT engine instances
- Consider document splitting
```

**Recovery Behavior:**

```
Auto-healing:
- Operation failure → Retry with same version → Automatic
- Network timeout → Client resends → Server deduplicates → Automatic
- Server crash → Operations persisted in Cassandra → Replay on restart

Human intervention:
- Persistent conflicts → Manual merge required
- Performance degradation → Optimize OT algorithm or scale
- Data corruption → Restore from snapshot + replay operations
```

---

## 1.6. CRDT (Conflict-Free Replicated Data Type) - Alternative Approach

### CRDT Fundamentals

**What is CRDT?**

CRDT (Conflict-Free Replicated Data Type) is an alternative to OT for collaborative editing. Unlike OT which requires a central server for transformation, CRDTs are designed to merge automatically without conflicts, making them ideal for peer-to-peer or decentralized architectures.

**Key Properties:**

1. **Commutativity**: Operations can be applied in any order
2. **Idempotency**: Applying the same operation multiple times has the same effect
3. **Associativity**: Operations can be grouped in any way

**Types of CRDTs:**

| Type | Description | Use Case |
|------|-------------|----------|
| **LSEQ** | Log-structured sequence | Text editing (character-by-character) |
| **RGA** | Replicated Growable Array | Text editing with insertions |
| **Yjs** | CRDT library with YATA algorithm | Full-featured collaborative editing |

### CRDT vs OT

| Aspect | OT | CRDT |
|--------|----|------|
| **Server Required** | Yes (for transformation) | No (peer-to-peer possible) |
| **Complexity** | High (transformation logic) | Medium (merge logic) |
| **Latency** | Low (server coordinates) | Very low (no server wait) |
| **Conflict Resolution** | Automatic (via transformation) | Automatic (via merge) |
| **Scalability** | Limited by server | Highly scalable |

### CRDT Implementation (Yjs-based)

```java
@Service
public class CRDTDocumentService {
    
    private final YDocumentStore documentStore;
    
    /**
     * Apply operation using CRDT merge
     */
    public void applyOperation(String docId, CRDTOperation op) {
        YDocument doc = documentStore.get(docId);
        
        // CRDT merge is automatic - no transformation needed
        doc.apply(op);
        
        // Persist merged state
        documentStore.save(doc);
        
        // Broadcast to other clients
        broadcastOperation(docId, op);
    }
    
    /**
     * Merge operations from multiple clients
     */
    public YDocument mergeOperations(String docId, List<CRDTOperation> ops) {
        YDocument doc = documentStore.get(docId);
        
        // CRDTs are commutative - apply in any order
        for (CRDTOperation op : ops) {
            doc.apply(op);
        }
        
        return doc;
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: CRDT Technology-Level

**Normal Flow: Concurrent Edits with CRDT**

```
Step 1: Two Users Edit Simultaneously
┌─────────────────────────────────────────────────────────────┐
│ Document: "Hello" (CRDT state)                             │
│ Alice types "A" at position 0                               │
│ Bob types "B" at position 0                                 │
│ Both operations sent to server (or peer-to-peer)           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Alice's Operation Applied Locally
┌─────────────────────────────────────────────────────────────┐
│ Alice's Client:                                              │
│ - Operation: insert("A", id=alice_1, timestamp=100)          │
│ - Apply locally: "Hello" → "AHello"                        │
│ - CRDT state updated with new character                     │
│ - Send to server/peers                                       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Bob's Operation Applied Locally
┌─────────────────────────────────────────────────────────────┐
│ Bob's Client:                                                │
│ - Operation: insert("B", id=bob_1, timestamp=101)           │
│ - Apply locally: "Hello" → "BHello"                        │
│ - CRDT state updated with new character                     │
│ - Send to server/peers                                       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Server Receives Both Operations
┌─────────────────────────────────────────────────────────────┐
│ Server:                                                      │
│ - Receives Alice's op: insert("A", id=alice_1, ts=100)      │
│ - Receives Bob's op: insert("B", id=bob_1, ts=101)          │
│                                                              │
│ CRDT Merge (automatic):                                      │
│ - Both operations have unique IDs                           │
│ - Operations are commutative                                │
│ - Merge by timestamp: "A" (ts=100) before "B" (ts=101)    │
│ - Result: "ABHello"                                         │
│                                                              │
│ No transformation needed!                                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Clients Receive and Merge
┌─────────────────────────────────────────────────────────────┐
│ Alice receives: insert("B", id=bob_1, ts=101)              │
│ - Current state: "AHello"                                   │
│ - Merge: Insert "B" after "A" (ts=101 > ts=100)            │
│ - Result: "ABHello"                                         │
│                                                              │
│ Bob receives: insert("A", id=alice_1, ts=100)              │
│ - Current state: "BHello"                                   │
│ - Merge: Insert "A" before "B" (ts=100 < ts=101)           │
│ - Result: "ABHello"                                         │
│                                                              │
│ Result: Both clients converge to "ABHello" automatically    │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Network Partition with CRDT**

```
Step 1: Network Partition
┌─────────────────────────────────────────────────────────────┐
│ T+0s: Document "Hello" (CRDT state)                         │
│ T+1s: Network partition                                     │
│   - Alice on partition A                                   │
│   - Bob on partition B                                      │
│   - Server on partition A                                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Independent Edits During Partition
┌─────────────────────────────────────────────────────────────┐
│ Partition A (Alice + Server):                               │
│ T+2s: Alice types "A" → "AHello"                           │
│ T+3s: Alice types "C" → "ACHello"                          │
│                                                              │
│ Partition B (Bob):                                          │
│ T+2s: Bob types "B" → "BHello"                             │
│ T+3s: Bob types "D" → "BDHello"                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Network Reconnects
┌─────────────────────────────────────────────────────────────┐
│ T+4s: Network reconnects                                     │
│                                                              │
│ Server receives operations from both partitions:            │
│ - From Alice: [insert("A", ts=100), insert("C", ts=101)]   │
│ - From Bob: [insert("B", ts=102), insert("D", ts=103)]     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: CRDT Merge (Automatic)
┌─────────────────────────────────────────────────────────────┐
│ Server CRDT Merge:                                           │
│ - All operations have unique IDs                            │
│ - Operations are commutative                                │
│ - Merge by timestamp order:                                 │
│   1. insert("A", ts=100) → "AHello"                        │
│   2. insert("C", ts=101) → "ACHello"                       │
│   3. insert("B", ts=102) → "ACBHello"                      │
│   4. insert("D", ts=103) → "ACBDHello"                     │
│                                                              │
│ Result: "ACBDHello" (deterministic merge)                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Clients Sync
┌─────────────────────────────────────────────────────────────┐
│ Alice syncs:                                                 │
│ - Receives: [insert("B", ts=102), insert("D", ts=103)]      │
│ - Current: "ACHello"                                        │
│ - Merge: "ACBDHello"                                        │
│                                                              │
│ Bob syncs:                                                   │
│ - Receives: [insert("A", ts=100), insert("C", ts=101)]      │
│ - Current: "BDHello"                                        │
│ - Merge: "ACBDHello"                                        │
│                                                              │
│ Result: All clients converge to same state                 │
└─────────────────────────────────────────────────────────────┘
```

**Idempotency Handling:**

```
CRDT Idempotency:
- Each operation has unique ID (client_id + sequence)
- Applying same operation multiple times: No effect
- Example:
  - Operation: insert("A", id=alice_1)
  - Apply once: "Hello" → "AHello"
  - Apply again: "AHello" → "AHello" (no change)
  - Idempotent by design
```

**Traffic Spike Handling:**

```
Scenario: 1000 users editing same document simultaneously

CRDT Handling:
- No central coordination needed
- Each operation has unique ID
- Operations merge automatically
- No conflicts possible
- Performance: O(n) where n = number of operations

Example:
- 1000 users type simultaneously
- 1000 operations generated
- All operations merge automatically
- Result: Deterministic final state
- No manual conflict resolution needed
```

**Recovery Behavior:**

```
CRDT Recovery:
- Operations are stored with unique IDs
- On reconnect: Send all operations since last sync
- Server merges all operations automatically
- Clients receive merged state
- No data loss (all operations preserved)

Example:
- Client offline for 1 hour
- 1000 operations occurred during offline
- On reconnect: Send all 1000 operations
- Server merges all operations
- Client receives final merged state
- Convergence: Automatic
```

---

## 2. Redis Deep Dive: Real-time State

### Why Redis for Collaborative Editing?

```
Problem: Need sub-millisecond access to document state

Requirements:
├── 50M active documents
├── State access for every operation
├── Presence updates every 5 seconds
├── < 1ms read latency

Redis provides:
├── In-memory storage
├── Sub-millisecond latency
├── Pub/sub for broadcasts
├── Data structures (hashes, sets)
└── Cluster for scale
```

### Data Structures

```java
@Service
public class DocumentStateCache {
    
    private final RedisTemplate<String, Object> redis;
    
    // Document state keys
    private String stateKey(String docId) {
        return "doc:state:" + docId;
    }
    
    private String presenceKey(String docId) {
        return "doc:presence:" + docId;
    }
    
    private String cursorKey(String docId, String odId) {
        return "doc:cursor:" + docId + ":" odId;
    }
    
    private String opsBufferKey(String docId) {
        return "doc:ops:" + docId;
    }
    
    /**
     * Get document state
     */
    public DocumentState getState(String documentId) {
        Map<Object, Object> state = redis.opsForHash()
            .entries(stateKey(documentId));
        
        if (state.isEmpty()) {
            return null;
        }
        
        return DocumentState.builder()
            .documentId(documentId)
            .version(Long.parseLong((String) state.get("version")))
            .content((String) state.get("content"))
            .lastModified(Instant.parse((String) state.get("lastModified")))
            .build();
    }
    
    /**
     * Update document state atomically
     */
    public void updateState(String documentId, long newVersion, String content) {
        String key = stateKey(documentId);
        
        // Use transaction for atomicity
        redis.execute(new SessionCallback<Object>() {
            @Override
            public Object execute(RedisOperations operations) {
                operations.multi();
                operations.opsForHash().put(key, "version", String.valueOf(newVersion));
                operations.opsForHash().put(key, "content", content);
                operations.opsForHash().put(key, "lastModified", Instant.now().toString());
                operations.expire(key, Duration.ofHours(2));
                return operations.exec();
            }
        });
    }
    
    /**
     * Update user presence
     */
    public void updatePresence(String documentId, String odId, UserPresence presence) {
        String presenceSetKey = presenceKey(documentId);
        String cursorHashKey = cursorKey(documentId, odId);
        
        // Add user to presence set with TTL
        redis.opsForSet().add(presenceSetKey, odId);
        redis.expire(presenceSetKey, Duration.ofMinutes(5));
        
        // Store cursor position
        redis.opsForHash().putAll(cursorHashKey, Map.of(
            "line", String.valueOf(presence.getCursor().getLine()),
            "ch", String.valueOf(presence.getCursor().getCh()),
            "selStart", String.valueOf(presence.getSelectionStart()),
            "selEnd", String.valueOf(presence.getSelectionEnd()),
            "color", presence.getColor(),
            "name", presence.getUserName()
        ));
        redis.expire(cursorHashKey, Duration.ofSeconds(30));
    }
    
    /**
     * Get all active users in document
     */
    public List<UserPresence> getPresence(String documentId) {
        Set<Object> userIds = redis.opsForSet().members(presenceKey(documentId));
        if (userIds == null || userIds.isEmpty()) {
            return Collections.emptyList();
        }
        
        List<UserPresence> presences = new ArrayList<>();
        for (Object odId : userIds) {
            Map<Object, Object> cursor = redis.opsForHash()
                .entries(cursorKey(documentId, (String) odId));
            if (!cursor.isEmpty()) {
                presences.add(UserPresence.fromMap((String) odId, cursor));
            }
        }
        return presences;
    }
}
```

### Redis Pub/Sub for Broadcasts

```java
@Service
public class DocumentBroadcaster {
    
    private final RedisTemplate<String, Object> redis;
    private final WebSocketSessionManager sessionManager;
    
    private String channel(String documentId) {
        return "doc:channel:" + documentId;
    }
    
    /**
     * Broadcast operation to all clients editing document
     */
    public void broadcastOperation(
            String documentId, 
            String senderClientId,
            long version, 
            Operation operation) {
        
        OperationMessage message = OperationMessage.builder()
            .type("operation")
            .documentId(documentId)
            .version(version)
            .operation(operation)
            .senderClientId(senderClientId)
            .timestamp(Instant.now())
            .build();
        
        redis.convertAndSend(channel(documentId), message);
    }
    
    /**
     * Broadcast presence update
     */
    public void broadcastPresence(String documentId, List<UserPresence> presences) {
        PresenceMessage message = PresenceMessage.builder()
            .type("presence")
            .documentId(documentId)
            .users(presences)
            .build();
        
        redis.convertAndSend(channel(documentId), message);
    }
    
    /**
     * Subscribe to document channel
     */
    @PostConstruct
    public void setupSubscription() {
        redis.getConnectionFactory().getConnection()
            .subscribe((message, pattern) -> {
                String channel = new String(message.getChannel());
                String documentId = channel.replace("doc:channel:", "");
                
                Object payload = redis.getValueSerializer()
                    .deserialize(message.getBody());
                
                if (payload instanceof OperationMessage) {
                    handleOperationMessage(documentId, (OperationMessage) payload);
                } else if (payload instanceof PresenceMessage) {
                    handlePresenceMessage(documentId, (PresenceMessage) payload);
                }
            }, "doc:channel:*".getBytes());
    }
    
    private void handleOperationMessage(String documentId, OperationMessage message) {
        // Get all WebSocket sessions for this document
        Set<WebSocketSession> sessions = sessionManager
            .getSessionsForDocument(documentId);
        
        for (WebSocketSession session : sessions) {
            String clientId = sessionManager.getClientId(session);
            
            // Don't send back to sender (they already have it)
            if (!clientId.equals(message.getSenderClientId())) {
                session.sendMessage(new TextMessage(toJson(message)));
            }
        }
    }
}
```

### Cache Stampede Prevention (Redis Document State Cache)

**What is Cache Stampede?**

Cache stampede occurs when cached document state expires and many requests simultaneously try to regenerate it, overwhelming the backend services.

**Scenario:**
```
T+0s:   Popular document's state cache expires
T+0s:   10,000 requests arrive simultaneously
T+0s:   All 10,000 requests see cache MISS
T+0s:   All 10,000 requests hit database/services simultaneously
T+1s:   Backend overwhelmed, latency spikes
```

**Prevention Strategy: Distributed Lock + Double-Check Pattern**

This system uses distributed locking to ensure only one request regenerates the cache:

1. **Distributed locking**: Only one thread fetches from database on cache miss
2. **Double-check pattern**: Re-check cache after acquiring lock
3. **TTL jitter**: Add random jitter to TTL to prevent simultaneous expiration

**Implementation:**

```java
@Service
public class DocumentStateCacheService {
    
    private final RedisTemplate<String, DocumentState> redis;
    private final DistributedLock lockService;
    private final DocumentRepository documentRepository;
    
    public DocumentState getCachedState(String documentId) {
        String key = "doc:state:" + documentId;
        
        // 1. Try cache first
        DocumentState cached = redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:doc:state:" + documentId;
        boolean acquired = lockService.tryLock(lockKey, Duration.ofSeconds(5));
        
        if (!acquired) {
            // Another thread is fetching, wait briefly
            try {
                Thread.sleep(50);
                cached = redis.opsForValue().get(key);
                if (cached != null) {
                    return cached;
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            // Fall back to database (acceptable degradation)
        }
        
        try {
            // 3. Double-check cache (might have been populated while acquiring lock)
            cached = redis.opsForValue().get(key);
            if (cached != null) {
                return cached;
            }
            
            // 4. Fetch from database (only one thread does this)
            DocumentState state = documentRepository.getState(documentId);
            
            if (state != null) {
                // 5. Populate cache with TTL jitter (add ±10% random jitter)
                Duration baseTtl = Duration.ofMinutes(5);
                long jitter = (long)(baseTtl.getSeconds() * 0.1 * (Math.random() * 2 - 1));
                Duration ttl = baseTtl.plusSeconds(jitter);
                
                redis.opsForValue().set(key, state, ttl);
            }
            
            return state;
            
        } finally {
            if (acquired) {
                lockService.unlock(lockKey);
            }
        }
    }
}
```

**TTL Jitter Benefits:**

- Prevents simultaneous expiration of related cache entries
- Reduces cache stampede probability
- Smooths out cache refresh load over time

---

## 3. Cassandra Deep Dive: Operations Log

### Why Cassandra for Operations?

```
Problem: Store billions of operations with efficient time-range queries

Requirements:
├── 300B operations/day
├── Time-range queries per document
├── High write throughput
├── Horizontal scaling

Cassandra provides:
├── Write-optimized (LSM trees)
├── Time-series friendly
├── Linear horizontal scaling
├── Tunable consistency
└── TTL for automatic cleanup
```

### Schema Design

```sql
-- Operations by document (main table)
CREATE TABLE operations_by_document (
    document_id UUID,
    version BIGINT,
    operation_id TIMEUUID,
    user_id UUID,
    operation_type TEXT,
    operation_data BLOB,
    created_at TIMESTAMP,
    PRIMARY KEY ((document_id), version, operation_id)
) WITH CLUSTERING ORDER BY (version ASC, operation_id ASC)
  AND default_time_to_live = 2592000  -- 30 days
  AND compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'compaction_window_unit': 'DAYS',
    'compaction_window_size': 1
  };

-- Operations by user (for audit)
CREATE TABLE operations_by_user (
    user_id UUID,
    day DATE,
    operation_id TIMEUUID,
    document_id UUID,
    operation_type TEXT,
    PRIMARY KEY ((user_id, day), operation_id)
) WITH CLUSTERING ORDER BY (operation_id DESC)
  AND default_time_to_live = 7776000;  -- 90 days

-- Version snapshots (for efficient loading)
CREATE TABLE version_snapshots (
    document_id UUID,
    version BIGINT,
    content BLOB,
    created_at TIMESTAMP,
    PRIMARY KEY ((document_id), version)
) WITH CLUSTERING ORDER BY (version DESC);
```

### Operations Repository

```java
@Repository
public class CassandraOperationRepository {
    
    private final CqlSession session;
    private final PreparedStatement insertOp;
    private final PreparedStatement selectOps;
    private final PreparedStatement selectOpsRange;
    
    @PostConstruct
    public void init() {
        insertOp = session.prepare(
            "INSERT INTO operations_by_document " +
            "(document_id, version, operation_id, user_id, operation_type, operation_data, created_at) " +
            "VALUES (?, ?, now(), ?, ?, ?, toTimestamp(now()))"
        );
        
        selectOps = session.prepare(
            "SELECT * FROM operations_by_document " +
            "WHERE document_id = ? AND version > ?"
        );
        
        selectOpsRange = session.prepare(
            "SELECT * FROM operations_by_document " +
            "WHERE document_id = ? AND version >= ? AND version <= ?"
        );
    }
    
    public void save(PersistedOperation op) {
        session.execute(insertOp.bind(
            op.getDocumentId(),
            op.getVersion(),
            op.getUserId(),
            op.getType().name(),
            serializeOperation(op.getOperation())
        ));
    }
    
    public List<Operation> getOperationsSince(UUID documentId, long sinceVersion) {
        ResultSet rs = session.execute(selectOps.bind(documentId, sinceVersion));
        
        List<Operation> operations = new ArrayList<>();
        for (Row row : rs) {
            operations.add(deserializeOperation(row.getByteBuffer("operation_data")));
        }
        return operations;
    }
    
    /**
     * Replay operations to rebuild document state
     */
    public Document replayOperations(UUID documentId, long fromVersion, long toVersion) {
        // Get base snapshot
        Document doc = getSnapshot(documentId, fromVersion);
        
        // Get operations in range
        ResultSet rs = session.execute(selectOpsRange.bind(
            documentId, fromVersion + 1, toVersion
        ));
        
        // Apply each operation
        for (Row row : rs) {
            Operation op = deserializeOperation(row.getByteBuffer("operation_data"));
            doc = applyOperation(doc, op);
        }
        
        return doc;
    }
}
```

---

## 4. WebSocket Deep Dive

### Connection Management

```java
@Component
public class DocumentWebSocketHandler extends TextWebSocketHandler {
    
    private final ConcurrentHashMap<String, Set<WebSocketSession>> documentSessions;
    private final ConcurrentHashMap<String, DocumentConnection> connections;
    private final OTEngine otEngine;
    private final DocumentStateCache stateCache;
    
    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        String documentId = extractDocumentId(session);
        String clientId = UUID.randomUUID().toString();
        
        // Store connection metadata
        connections.put(session.getId(), new DocumentConnection(
            session, documentId, clientId, null, Instant.now()
        ));
        
        // Add to document sessions
        documentSessions.computeIfAbsent(documentId, k -> ConcurrentHashMap.newKeySet())
            .add(session);
        
        log.info("WebSocket connected: doc={}, client={}", documentId, clientId);
    }
    
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        DocumentConnection conn = connections.get(session.getId());
        if (conn == null) {
            return;
        }
        
        try {
            ClientMessage msg = parseMessage(message.getPayload());
            
            switch (msg.getType()) {
                case "auth":
                    handleAuth(session, conn, (AuthMessage) msg);
                    break;
                case "operation":
                    handleOperation(session, conn, (OperationMessage) msg);
                    break;
                case "cursor":
                    handleCursor(session, conn, (CursorMessage) msg);
                    break;
                case "ping":
                    handlePing(session, conn);
                    break;
            }
        } catch (Exception e) {
            log.error("Error handling message", e);
            sendError(session, "internal_error", e.getMessage());
        }
    }
    
    private void handleAuth(WebSocketSession session, DocumentConnection conn, AuthMessage msg) {
        // Validate JWT token
        User user = authService.validateToken(msg.getToken());
        if (user == null) {
            sendError(session, "auth_failed", "Invalid token");
            session.close();
            return;
        }
        
        // Check document access
        if (!permissionService.canAccess(user.getId(), conn.getDocumentId())) {
            sendError(session, "access_denied", "No access to document");
            session.close();
            return;
        }
        
        // Update connection with user info
        conn.setUser(user);
        conn.setLastVersion(msg.getLastVersion());
        
        // Load document state
        DocumentState state = stateCache.getState(conn.getDocumentId());
        if (state == null) {
            state = loadDocumentFromStorage(conn.getDocumentId());
        }
        
        // Send initial state
        AuthSuccessMessage response = AuthSuccessMessage.builder()
            .type("auth_success")
            .userId(user.getId())
            .documentVersion(state.getVersion())
            .build();
        
        // Include missed operations if client has older version
        if (msg.getLastVersion() < state.getVersion()) {
            List<Operation> missed = operationRepo.getOperationsSince(
                conn.getDocumentId(), msg.getLastVersion()
            );
            response.setMissedOperations(missed);
        }
        
        // Include current presence
        response.setActiveUsers(stateCache.getPresence(conn.getDocumentId()));
        
        sendMessage(session, response);
        
        // Update presence
        updatePresence(conn);
    }
    
    private void handleOperation(WebSocketSession session, DocumentConnection conn, OperationMessage msg) {
        // Process through OT engine
        OperationResult result = otEngine.processOperation(
            conn.getDocumentId(),
            conn.getClientId(),
            msg.getVersion(),
            msg.getOperation()
        );
        
        // Send acknowledgment to sender
        sendMessage(session, AckMessage.builder()
            .type("ack")
            .clientId(conn.getClientId())
            .version(result.getVersion())
            .build()
        );
        
        // Broadcast is handled by OT engine via Redis pub/sub
    }
    
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        DocumentConnection conn = connections.remove(session.getId());
        if (conn != null) {
            documentSessions.get(conn.getDocumentId()).remove(session);
            
            // Remove from presence
            stateCache.removePresence(conn.getDocumentId(), conn.getUserId());
            
            // Broadcast presence update
            broadcaster.broadcastPresence(
                conn.getDocumentId(),
                stateCache.getPresence(conn.getDocumentId())
            );
        }
    }
}
```

### Connection Scaling

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    WebSocket Connection Scaling                               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Challenge: 30M concurrent WebSocket connections                              │
│                                                                               │
│  Solution: Distributed WebSocket servers with Redis coordination              │
│                                                                               │
│  Architecture:                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │                                                                      │     │
│  │   Client A ──┐                              ┌── Client C             │     │
│  │              │     ┌──────────────┐        │                        │     │
│  │   Client B ──┼────>│  WS Server 1 │<───────┼── Client D             │     │
│  │              │     └──────┬───────┘        │                        │     │
│  │              │            │                │                        │     │
│  │              │            │ Redis Pub/Sub  │                        │     │
│  │              │            │                │                        │     │
│  │   Client E ──┼────>┌──────▼───────┐<───────┼── Client G             │     │
│  │              │     │  WS Server 2 │        │                        │     │
│  │   Client F ──┘     └──────────────┘        └── Client H             │     │
│  │                                                                      │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  Sticky Sessions:                                                             │
│  ├── Load balancer routes by document_id                                     │
│  ├── All clients for same document → same server                            │
│  ├── Reduces cross-server communication                                      │
│  └── Fallback to Redis pub/sub if needed                                    │
│                                                                               │
│  Server Capacity:                                                             │
│  ├── 50K connections per server                                              │
│  ├── 600 servers for 30M connections                                         │
│  ├── 8 vCPU, 32 GB RAM per server                                           │
│  └── Memory: ~500 bytes per connection                                       │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Step-by-Step Simulations

### Simulation 1: Two Users Typing Simultaneously

```
Scenario: Alice and Bob both type at position 0 in "Hello"

Initial State:
├── Document: "Hello"
├── Version: 10
├── Alice's cursor: position 0
├── Bob's cursor: position 0

Step 1: Both users type (T=0ms)
├── Alice types "A" at position 0 (local)
├── Bob types "B" at position 0 (local)
├── Alice sees: "AHello"
├── Bob sees: "BHello"

Step 2: Operations sent to server (T=50ms)
├── Alice sends: insert("A", 0), version=10
├── Bob sends: insert("B", 0), version=10
├── Server receives Alice first (network timing)

Step 3: Server processes Alice's operation (T=55ms)
├── No transformation needed (first op)
├── Apply: "Hello" → "AHello"
├── New version: 11
├── Broadcast to Bob

Step 4: Server processes Bob's operation (T=60ms)
├── Transform against Alice's op:
│   ├── Original: insert("B", 0)
│   ├── Alice inserted at 0, length 1
│   ├── Transformed: insert("B", 1)
├── Apply: "AHello" → "ABHello"
├── New version: 12
├── Broadcast to Alice

Step 5: Alice receives Bob's transformed op (T=100ms)
├── Receives: insert("B", 1), version=12
├── Already has "AHello"
├── Apply: "AHello" → "ABHello"
├── Alice sees: "ABHello"

Step 6: Bob receives Alice's op and his ACK (T=110ms)
├── Receives: insert("A", 0), version=11
├── Has pending op: insert("B", 0)
├── Transform pending: insert("B", 1)
├── Apply Alice's: "BHello" → "ABHello"
├── Bob sees: "ABHello"

Final State:
├── Document: "ABHello"
├── Version: 12
├── Both users see same content
├── Both users' intentions preserved
```

### Simulation 2: Offline Editing and Sync

```
Scenario: Alice goes offline, edits, then reconnects

Initial State:
├── Document: "The quick brown fox"
├── Version: 20
├── Alice: Online, editing
├── Bob: Online, editing

Step 1: Alice goes offline (T=0)
├── Alice's last version: 20
├── Alice continues editing locally
├── Operations buffered locally

Step 2: Alice edits offline (T=0-5min)
├── Op1: insert(" lazy", 9) → "The quick lazy brown fox"
├── Op2: delete(20, 4) → "The quick lazy brown"
├── Local buffer: [Op1, Op2]

Step 3: Bob edits online (T=1min)
├── Op: insert("very ", 4) → "The very quick brown fox"
├── Server version: 21

Step 4: Alice reconnects (T=5min)
├── Sends: reconnect, last_version=20
├── Server sends: ops since v20 = [Bob's insert]

Step 5: Alice transforms local ops
├── Bob's op: insert("very ", 4)
├── Transform Op1:
│   ├── Original: insert(" lazy", 9)
│   ├── Bob inserted at 4, length 5
│   ├── 9 > 4, so shift: insert(" lazy", 14)
├── Transform Op2:
│   ├── Original: delete(20, 4)
│   ├── Shift: delete(25, 4)

Step 6: Alice sends transformed ops
├── Server receives transformed ops
├── Applies to "The very quick brown fox"
├── Result: "The very quick lazy brown"
├── Version: 23

Step 7: Convergence
├── Alice: "The very quick lazy brown"
├── Bob receives Alice's ops
├── Bob: "The very quick lazy brown"
├── Both at version 23
```

