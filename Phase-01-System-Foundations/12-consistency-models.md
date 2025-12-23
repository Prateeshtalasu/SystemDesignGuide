# Consistency Models

## 0️⃣ Prerequisites

Before diving into consistency models, you should understand:

- **Distributed Systems** (covered in Topic 01): Systems where multiple computers work together over a network
- **CAP Theorem** (covered in Topic 06): The tradeoff between Consistency, Availability, and Partition Tolerance
- **Replication**: The concept of keeping copies of data on multiple servers (we'll explain this in detail below)

**Quick refresher on replication**: When you have important data, you don't want it on just one server (what if that server dies?). So you copy it to multiple servers. These copies are called "replicas." The challenge is: when you update data on one replica, how and when do the other replicas get the update?

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

Imagine you're building a banking app. A user has $1,000 in their account. They make two transactions:

1. **Device A (phone)**: Withdraw $600
2. **Device B (laptop)**: Check balance

If Device B sees the old balance ($1,000) after the withdrawal happened, the user might think they have more money than they do. They might overdraw their account.

Now imagine this at scale: millions of users, thousands of servers, data replicated across continents. How do you ensure everyone sees the "right" data at the "right" time?

### What Systems Looked Like Before Consistency Models Were Formalized

In the early days of distributed systems:

1. **Ad-hoc solutions**: Each team invented their own way to handle data consistency
2. **Hidden bugs**: Systems would work most of the time, then fail mysteriously under load
3. **No vocabulary**: Engineers couldn't communicate about tradeoffs because there were no standard terms
4. **Over-engineering**: Some systems used expensive coordination when cheaper options would work

### What Breaks Without Clear Consistency Models

**Without understanding consistency:**

- **Lost updates**: Two users edit the same document, one edit disappears
- **Dirty reads**: User sees partially written data
- **Stale reads**: User sees old data after an update
- **Phantom reads**: Query results change between reads
- **Split brain**: Different parts of the system disagree on the current state

### Real Examples of the Problem

**Amazon Shopping Cart (Famous Case Study)**:
In the early 2000s, Amazon discovered that using strong consistency for shopping carts was causing availability problems. If the database was temporarily unreachable, users couldn't add items to their cart. They realized that for shopping carts, it's better to occasionally have duplicate items (which users can remove) than to have the cart be unavailable. This led to Amazon's Dynamo paper and the concept of "eventual consistency."

**Google Spanner**:
Google needed a globally distributed database for AdWords (billions of dollars in transactions). They couldn't tolerate inconsistency (imagine charging advertisers twice or not at all). They built Spanner, which provides strong consistency globally, but at the cost of higher latency.

---

## 2️⃣ Intuition and Mental Model

### The Library Book Analogy

Imagine a library system with multiple branches across a city. Each branch has a catalog showing which books are available.

**Scenario 1: Strong Consistency (Single Source of Truth)**
- There's ONE master catalog at the central library
- Every branch must call the central library before lending a book
- Slow (you wait on hold) but always accurate
- If the phone line is down, no one can borrow books

**Scenario 2: Eventual Consistency (Local Copies)**
- Each branch has its own catalog copy
- When a book is borrowed, the branch updates its copy
- Updates are sent to other branches overnight
- Fast (no waiting) but you might go to a branch for a book that was just borrowed elsewhere
- Even if phone lines are down, you can still borrow books

**Scenario 3: Causal Consistency (Smart Updates)**
- Updates that are related are sent together
- If you return a book and immediately reserve it for a friend, both updates travel together
- Unrelated updates can happen in any order

This analogy will help us understand the different consistency models.

### The Consistency Spectrum

```
STRONGER ←──────────────────────────────────────────→ WEAKER
                                                      
Linearizable → Sequential → Causal → Eventual → No guarantee
                                                      
More coordination                    Less coordination
Higher latency                       Lower latency
Lower availability                   Higher availability
Easier to reason about               Harder to reason about
```

---

## 3️⃣ How It Works Internally

### Understanding the Core Concepts

#### What is a "Read" and "Write"?

- **Write**: Storing or updating data (e.g., `balance = 500`)
- **Read**: Retrieving data (e.g., `get balance`)

#### What is "Consistency"?

Consistency defines **what values a read operation can return** given the writes that have happened.

Different consistency models make different promises about this.

---

### The Consistency Models Explained

#### 1. Strong Consistency (Linearizability)

**Definition**: Every read returns the most recent write. All operations appear to happen in a single, global order that matches real-time.

**How it works internally**:

```
Time ──────────────────────────────────────────────────→

Writer:     ┌─────WRITE(x=1)─────┐
            │                     │
            start               commit

Reader 1:                              ┌──READ(x)──┐
                                       │           │
                                       Returns: 1 ✓

Reader 2:           ┌──READ(x)──┐
                    │           │
                    Must wait until write completes OR return old value
```

**Implementation mechanism**:
1. All writes go through a single leader (or use consensus like Paxos/Raft)
2. Writes are not acknowledged until replicated to a majority
3. Reads either go to the leader or use quorum reads

```java
// Pseudocode for linearizable read
public int linearizableRead(String key) {
    // Option 1: Always read from leader
    return leader.read(key);
    
    // Option 2: Quorum read (read from majority)
    List<Integer> values = new ArrayList<>();
    for (Node replica : replicas) {
        values.add(replica.read(key));
    }
    // Return value from majority (with highest version)
    return getMajorityValue(values);
}
```

**Real-world example**: Google Spanner, CockroachDB, single-node databases

---

#### 2. Sequential Consistency

**Definition**: All operations appear to happen in some sequential order, and each process's operations appear in the order they were issued. But this order doesn't have to match real-time.

**The difference from linearizability**:

```
Real time:
Process A: WRITE(x=1) ─────────────────────────────────
Process B: ─────────────────── READ(x) ────────────────

Linearizable: READ must return 1 (respects real-time order)
Sequential:   READ can return 0 OR 1 (just needs SOME consistent order)
```

**Why would you want this?**
- Cheaper to implement than linearizability
- Still provides strong guarantees for single-process reasoning

---

#### 3. Causal Consistency

**Definition**: Operations that are causally related are seen by all processes in the same order. Concurrent operations (not causally related) can be seen in different orders.

**What is "causally related"?**
- If Process A writes X, then reads X, then writes Y, the write to Y is causally related to the write to X
- If Process A writes X, and Process B (without reading X) writes Y, they are NOT causally related

**Visual example**:

```
Process A: WRITE(x=1) ──────────────────────────────────
                ↓ (A reads x, then writes y)
Process A: ─────────────── WRITE(y=2) ──────────────────

Process B: ──────────────────────────── READ(y) → READ(x)
                                        
Causal guarantee: If B sees y=2, B MUST see x=1
                  (because y=2 was caused by x=1)
```

**How it works internally**:
- Each write carries a "version vector" or "logical clock"
- When you read a value, you get its causal dependencies
- When you write, you include what you've seen

```java
public class CausalStore {
    // Version vector tracks what each node has seen
    private Map<String, Integer> versionVector = new HashMap<>();
    
    public void write(String key, Object value, Map<String, Integer> clientVector) {
        // Merge client's vector with ours
        for (Map.Entry<String, Integer> entry : clientVector.entrySet()) {
            versionVector.merge(entry.getKey(), entry.getValue(), Math::max);
        }
        // Increment our own version
        versionVector.merge(nodeId, 1, Integer::sum);
        // Store value with version vector
        store.put(key, new VersionedValue(value, new HashMap<>(versionVector)));
    }
    
    public VersionedValue read(String key) {
        // Return value with its version vector
        // Client will use this vector for future writes
        return store.get(key);
    }
}
```

**Real-world example**: MongoDB (with causal consistency sessions), COPS system

---

#### 4. Eventual Consistency

**Definition**: If no new updates are made, eventually all replicas will converge to the same value. No guarantees about what you'll read in the meantime.

**How it works internally**:

```
Time ──────────────────────────────────────────────────→

Node A: WRITE(x=1) ───────────────────────────────────→
                    │
                    └──── async replication ────┐
                                                ↓
Node B: ──────────────────────────────────── x=1 (eventually)

Reader at Node B (before replication): READ(x) → 0 (stale!)
Reader at Node B (after replication):  READ(x) → 1 (correct)
```

**The "eventually" part**:
- Could be milliseconds (same datacenter)
- Could be seconds (cross-datacenter)
- Could be longer if there are network issues

**How replication typically works**:

```java
public class EventuallyConsistentStore {
    private Map<String, VersionedValue> localStore = new ConcurrentHashMap<>();
    private List<Node> replicas;
    
    public void write(String key, Object value) {
        // Write locally immediately
        long version = System.currentTimeMillis();
        localStore.put(key, new VersionedValue(value, version));
        
        // Asynchronously replicate to other nodes
        CompletableFuture.runAsync(() -> {
            for (Node replica : replicas) {
                try {
                    replica.replicate(key, value, version);
                } catch (Exception e) {
                    // Will retry later via anti-entropy
                    pendingReplications.add(new Replication(replica, key, value, version));
                }
            }
        });
    }
    
    public Object read(String key) {
        // Read locally, might be stale
        VersionedValue vv = localStore.get(key);
        return vv != null ? vv.value : null;
    }
}
```

**Real-world example**: Amazon DynamoDB (default), Cassandra, DNS

---

#### 5. Read-Your-Writes Consistency

**Definition**: A process will always see its own writes. Other processes might not.

**Why this matters**:

```
Without read-your-writes:
User: POST "Hello World" ──→ Server A (writes to DB)
User: GET posts ──→ Server B (reads from replica)
User sees: Nothing! (replica hasn't caught up)
User: "Where's my post?!" 😡
```

**How to implement**:

```java
public class ReadYourWritesStore {
    
    public void write(String userId, String key, Object value) {
        long version = writeToMaster(key, value);
        // Remember what version this user has written
        userWriteVersions.put(userId, key, version);
    }
    
    public Object read(String userId, String key) {
        Long minVersion = userWriteVersions.get(userId, key);
        
        if (minVersion != null) {
            // This user has written to this key
            // Make sure we read at least that version
            return readWithMinVersion(key, minVersion);
        } else {
            // User hasn't written, any replica is fine
            return readFromAnyReplica(key);
        }
    }
}
```

**Real-world example**: Facebook's TAO, most social media platforms

---

#### 6. Monotonic Reads

**Definition**: If a process reads a value, subsequent reads will never return an older value.

**Why this matters**:

```
Without monotonic reads:
User: GET balance ──→ Server A ──→ Returns: $100
User: GET balance ──→ Server B ──→ Returns: $80 (older!)
User: "Wait, where did my $20 go?!" 😱
```

**How to implement**:

```java
public class MonotonicReadStore {
    // Track the version each user has seen
    private Map<String, Map<String, Long>> userSeenVersions = new ConcurrentHashMap<>();
    
    public VersionedValue read(String userId, String key) {
        Long lastSeenVersion = userSeenVersions
            .getOrDefault(userId, Collections.emptyMap())
            .get(key);
        
        VersionedValue value;
        if (lastSeenVersion != null) {
            // Read from replica that has at least this version
            value = readWithMinVersion(key, lastSeenVersion);
        } else {
            value = readFromAnyReplica(key);
        }
        
        // Update what this user has seen
        userSeenVersions
            .computeIfAbsent(userId, k -> new ConcurrentHashMap<>())
            .put(key, value.version);
        
        return value;
    }
}
```

---

## 4️⃣ Simulation-First Explanation

Let's trace through a concrete example to understand how different consistency models behave.

### Scenario: Social Media "Like" Counter

**Setup**:
- A post has 100 likes
- Two users (Alice and Bob) both click "like" at almost the same time
- The system has 3 replicas (R1, R2, R3)

### With Strong Consistency (Linearizability)

```
Initial state: likes = 100 (on all replicas)

Time 0ms:   Alice clicks like, request goes to R1
Time 1ms:   Bob clicks like, request goes to R2

R1 receives Alice's like:
  1. R1 acquires distributed lock (or becomes leader for this key)
  2. R1 reads current value: 100
  3. R1 increments: 101
  4. R1 replicates to R2 and R3 (waits for acknowledgment)
  5. R1 releases lock
  6. R1 responds to Alice: "Like recorded! Total: 101"
  
Time 50ms: Replication complete

R2 receives Bob's like:
  1. R2 acquires distributed lock (waits if R1 has it)
  2. R2 reads current value: 101
  3. R2 increments: 102
  4. R2 replicates to R1 and R3
  5. R2 releases lock
  6. R2 responds to Bob: "Like recorded! Total: 102"

Final state: likes = 102 (on all replicas) ✓
```

**Latency**: 50-100ms per like (due to coordination)
**Correctness**: Perfect, no lost likes

### With Eventual Consistency

```
Initial state: likes = 100 (on all replicas)

Time 0ms:   Alice clicks like, request goes to R1
Time 1ms:   Bob clicks like, request goes to R2

R1 receives Alice's like:
  1. R1 reads local value: 100
  2. R1 increments: 101
  3. R1 stores locally
  4. R1 responds to Alice: "Like recorded! Total: 101"
  5. R1 asynchronously sends update to R2, R3
  
R2 receives Bob's like (concurrently):
  1. R2 reads local value: 100 (hasn't received R1's update yet)
  2. R2 increments: 101
  3. R2 stores locally
  4. R2 responds to Bob: "Like recorded! Total: 101"
  5. R2 asynchronously sends update to R1, R3

Time 5ms: Both respond to users
Time 100ms: Updates propagate

Problem! R1 has 101, R2 has 101, but there were 2 likes!

Conflict resolution needed:
  - Last-write-wins: One like is lost! Final: 101 ✗
  - CRDT counter: Both likes preserved! Final: 102 ✓
```

**Latency**: 5ms per like (no coordination)
**Correctness**: Depends on conflict resolution strategy

### With CRDT (Conflict-free Replicated Data Type)

CRDTs are data structures designed for eventual consistency that automatically resolve conflicts.

```java
/**
 * G-Counter: A grow-only counter CRDT.
 * Each node maintains its own count, total is sum of all.
 */
public class GCounter {
    private final String nodeId;
    private final Map<String, Long> counts = new ConcurrentHashMap<>();
    
    public GCounter(String nodeId) {
        this.nodeId = nodeId;
        counts.put(nodeId, 0L);
    }
    
    public void increment() {
        counts.merge(nodeId, 1L, Long::sum);
    }
    
    public long value() {
        return counts.values().stream().mapToLong(Long::longValue).sum();
    }
    
    public void merge(GCounter other) {
        for (Map.Entry<String, Long> entry : other.counts.entrySet()) {
            counts.merge(entry.getKey(), entry.getValue(), Math::max);
        }
    }
}
```

```
With CRDT:

R1's counter: {R1: 1, R2: 0, R3: 0} → total = 1
R2's counter: {R1: 0, R2: 1, R3: 0} → total = 1

After merge:
All replicas: {R1: 1, R2: 1, R3: 0} → total = 2 ✓
```

---

## 5️⃣ How Engineers Actually Use This in Production

### At Major Companies

**Amazon (DynamoDB)**:
- Default: Eventual consistency (faster, cheaper)
- Option: Strong consistency (2x read cost, higher latency)
- Engineers choose per-query based on requirements

```java
// DynamoDB SDK example
GetItemRequest request = GetItemRequest.builder()
    .tableName("Users")
    .key(Map.of("userId", AttributeValue.builder().s("123").build()))
    .consistentRead(true)  // Strong consistency
    .build();
```

**Google (Spanner)**:
- Always strongly consistent (linearizable)
- Uses TrueTime (GPS + atomic clocks) for global ordering
- Higher latency but simpler programming model

**Facebook (TAO)**:
- Eventual consistency for most reads
- Read-your-writes for the user who made the change
- "Social graph doesn't need to be perfectly consistent"

**Netflix**:
- Cassandra with eventual consistency
- Uses client-side timestamps for conflict resolution
- Accepts that two users might briefly see different data

### Choosing the Right Model

```
┌─────────────────────────────────────────────────────────────┐
│                 CONSISTENCY DECISION TREE                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Is this financial/critical data?                           │
│  ├── YES → Use STRONG consistency                           │
│  │         (bank transfers, inventory counts)               │
│  │                                                          │
│  └── NO → Can users tolerate stale data?                    │
│           ├── NO → Use READ-YOUR-WRITES                     │
│           │        (user's own posts, settings)             │
│           │                                                 │
│           └── YES → Use EVENTUAL consistency                │
│                     (likes, views, recommendations)         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Real Production Patterns

#### Pattern 1: Strong for Writes, Eventual for Reads

```java
@Service
public class InventoryService {
    
    @Transactional  // Strong consistency for writes
    public void purchaseItem(String itemId, int quantity) {
        // This MUST be strongly consistent
        // We can't oversell inventory
        int available = inventoryRepo.getAvailableForUpdate(itemId);
        if (available < quantity) {
            throw new InsufficientInventoryException();
        }
        inventoryRepo.decrementInventory(itemId, quantity);
    }
    
    // Eventual consistency is fine for display
    @Cacheable("inventory-display")
    public int getDisplayInventory(String itemId) {
        // Can be slightly stale, it's just for display
        return inventoryRepo.getAvailable(itemId);
    }
}
```

#### Pattern 2: Session Consistency

```java
@Component
public class SessionConsistencyFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        // Get the last write timestamp from session
        Long lastWriteTime = (Long) httpRequest.getSession()
            .getAttribute("lastWriteTime");
        
        if (lastWriteTime != null) {
            // Ensure reads see at least this timestamp
            ReadConsistencyContext.setMinTimestamp(lastWriteTime);
        }
        
        chain.doFilter(request, response);
    }
}
```

---

## 6️⃣ How to Implement or Apply It

### Building a Simple Eventually Consistent Store

```java
package com.systemdesign.consistency;

import java.util.*;
import java.util.concurrent.*;

/**
 * A simple eventually consistent key-value store with anti-entropy.
 * Demonstrates the core concepts of eventual consistency.
 */
public class EventuallyConsistentStore {
    
    private final String nodeId;
    private final Map<String, VersionedValue> store = new ConcurrentHashMap<>();
    private final List<EventuallyConsistentStore> peers = new ArrayList<>();
    private final ScheduledExecutorService antiEntropyExecutor = 
        Executors.newSingleThreadScheduledExecutor();
    
    public EventuallyConsistentStore(String nodeId) {
        this.nodeId = nodeId;
        // Anti-entropy: periodically sync with peers
        antiEntropyExecutor.scheduleAtFixedRate(
            this::antiEntropy, 1, 1, TimeUnit.SECONDS);
    }
    
    public void addPeer(EventuallyConsistentStore peer) {
        peers.add(peer);
    }
    
    /**
     * Write a value. Uses wall-clock time as version.
     * In production, you'd use vector clocks or hybrid logical clocks.
     */
    public void put(String key, String value) {
        long version = System.currentTimeMillis();
        VersionedValue vv = new VersionedValue(value, version, nodeId);
        store.put(key, vv);
        
        // Asynchronously propagate to peers
        for (EventuallyConsistentStore peer : peers) {
            CompletableFuture.runAsync(() -> peer.receive(key, vv));
        }
    }
    
    /**
     * Read a value. Returns local value, which might be stale.
     */
    public String get(String key) {
        VersionedValue vv = store.get(key);
        return vv != null ? vv.value : null;
    }
    
    /**
     * Receive a replicated value from a peer.
     * Uses last-write-wins for conflict resolution.
     */
    public void receive(String key, VersionedValue incoming) {
        store.merge(key, incoming, (existing, newVal) -> {
            // Last-write-wins: keep the value with higher version
            if (newVal.version > existing.version) {
                return newVal;
            } else if (newVal.version == existing.version) {
                // Tie-breaker: use node ID
                return newVal.nodeId.compareTo(existing.nodeId) > 0 ? newVal : existing;
            }
            return existing;
        });
    }
    
    /**
     * Anti-entropy: periodically sync entire state with a random peer.
     * This handles missed updates due to network issues.
     */
    private void antiEntropy() {
        if (peers.isEmpty()) return;
        
        // Pick a random peer
        EventuallyConsistentStore peer = peers.get(
            ThreadLocalRandom.current().nextInt(peers.size()));
        
        // Send all our data
        for (Map.Entry<String, VersionedValue> entry : store.entrySet()) {
            peer.receive(entry.getKey(), entry.getValue());
        }
    }
    
    public Map<String, VersionedValue> getState() {
        return new HashMap<>(store);
    }
    
    public static class VersionedValue {
        final String value;
        final long version;
        final String nodeId;
        
        VersionedValue(String value, long version, String nodeId) {
            this.value = value;
            this.version = version;
            this.nodeId = nodeId;
        }
        
        @Override
        public String toString() {
            return value + "@" + version;
        }
    }
}
```

### Demonstration

```java
package com.systemdesign.consistency;

public class ConsistencyDemo {
    
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Eventual Consistency Demo ===\n");
        
        // Create 3 nodes
        EventuallyConsistentStore node1 = new EventuallyConsistentStore("node1");
        EventuallyConsistentStore node2 = new EventuallyConsistentStore("node2");
        EventuallyConsistentStore node3 = new EventuallyConsistentStore("node3");
        
        // Connect them
        node1.addPeer(node2);
        node1.addPeer(node3);
        node2.addPeer(node1);
        node2.addPeer(node3);
        node3.addPeer(node1);
        node3.addPeer(node2);
        
        // Write to node1
        System.out.println("Writing 'hello' to node1...");
        node1.put("greeting", "hello");
        
        // Immediately read from all nodes
        System.out.println("\nImmediate reads:");
        System.out.println("  node1: " + node1.get("greeting"));
        System.out.println("  node2: " + node2.get("greeting")); // Might be null!
        System.out.println("  node3: " + node3.get("greeting")); // Might be null!
        
        // Wait for propagation
        Thread.sleep(100);
        
        System.out.println("\nAfter 100ms:");
        System.out.println("  node1: " + node1.get("greeting"));
        System.out.println("  node2: " + node2.get("greeting")); // Should be "hello"
        System.out.println("  node3: " + node3.get("greeting")); // Should be "hello"
        
        // Concurrent writes (conflict scenario)
        System.out.println("\n--- Concurrent Write Scenario ---");
        System.out.println("node1 writes 'bonjour', node2 writes 'hola' simultaneously...");
        
        // Simulate concurrent writes
        node1.put("greeting", "bonjour");
        node2.put("greeting", "hola");
        
        Thread.sleep(100);
        
        System.out.println("\nAfter conflict resolution:");
        System.out.println("  node1: " + node1.get("greeting"));
        System.out.println("  node2: " + node2.get("greeting"));
        System.out.println("  node3: " + node3.get("greeting"));
        System.out.println("(All should converge to the same value due to last-write-wins)");
        
        System.exit(0);
    }
}
```

### Building Read-Your-Writes Consistency

```java
package com.systemdesign.consistency;

import java.util.*;
import java.util.concurrent.*;

/**
 * Demonstrates read-your-writes consistency on top of eventual consistency.
 */
public class ReadYourWritesStore {
    
    private final EventuallyConsistentStore[] replicas;
    
    // Track each user's last write version per key
    private final Map<String, Map<String, Long>> userWriteVersions = 
        new ConcurrentHashMap<>();
    
    public ReadYourWritesStore(int replicaCount) {
        replicas = new EventuallyConsistentStore[replicaCount];
        for (int i = 0; i < replicaCount; i++) {
            replicas[i] = new EventuallyConsistentStore("replica-" + i);
        }
        // Connect replicas
        for (EventuallyConsistentStore replica : replicas) {
            for (EventuallyConsistentStore other : replicas) {
                if (replica != other) {
                    replica.addPeer(other);
                }
            }
        }
    }
    
    /**
     * Write on behalf of a user. Tracks the write version.
     */
    public void put(String userId, String key, String value) {
        long version = System.currentTimeMillis();
        
        // Write to a replica
        EventuallyConsistentStore replica = pickReplica(key);
        replica.put(key, value);
        
        // Remember this user's write version
        userWriteVersions
            .computeIfAbsent(userId, k -> new ConcurrentHashMap<>())
            .put(key, version);
    }
    
    /**
     * Read on behalf of a user. Ensures they see their own writes.
     */
    public String get(String userId, String key) {
        Long minVersion = userWriteVersions
            .getOrDefault(userId, Collections.emptyMap())
            .get(key);
        
        if (minVersion == null) {
            // User hasn't written to this key, any replica is fine
            return pickReplica(key).get(key);
        }
        
        // User has written, find a replica with at least that version
        for (EventuallyConsistentStore replica : replicas) {
            EventuallyConsistentStore.VersionedValue vv = 
                replica.getState().get(key);
            if (vv != null && vv.version >= minVersion) {
                return vv.value;
            }
        }
        
        // Fallback: read from any replica
        // In production, you might wait or route to the write replica
        return pickReplica(key).get(key);
    }
    
    private EventuallyConsistentStore pickReplica(String key) {
        // Simple hash-based routing
        int index = Math.abs(key.hashCode()) % replicas.length;
        return replicas[index];
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

#### 1. Assuming Strong Consistency When You Have Eventual

**Wrong assumption:**
```java
// This code assumes strong consistency
public void transferMoney(String fromAccount, String toAccount, int amount) {
    int fromBalance = accountStore.get(fromAccount);  // Might be stale!
    if (fromBalance >= amount) {
        accountStore.put(fromAccount, fromBalance - amount);
        accountStore.put(toAccount, 
            accountStore.get(toAccount) + amount);  // Race condition!
    }
}
```

**With eventual consistency, this can:**
- Read stale balances
- Allow overdrafts
- Lose money during concurrent transfers

#### 2. Ignoring the "Eventually" in Eventual Consistency

**Wrong assumption:**
```java
// Write then immediately read
userStore.put("user123", updatedUser);
User user = userStore.get("user123");  // Might return OLD user!
return user;  // Bug: returning stale data
```

**Fix:**
```java
// Option 1: Return what you wrote
userStore.put("user123", updatedUser);
return updatedUser;  // Return the value you just wrote

// Option 2: Use read-your-writes consistency
userStore.putWithSession(sessionId, "user123", updatedUser);
return userStore.getWithSession(sessionId, "user123");
```

#### 3. Using Wall-Clock Time for Ordering

**Problem:** Different servers have different clocks.

```
Server A's clock: 10:00:00.000
Server B's clock: 10:00:00.500 (500ms ahead)

Server A writes at its 10:00:00.100 → timestamp 100
Server B writes at its 10:00:00.200 → timestamp 700

Last-write-wins picks Server B, even though Server A wrote later in real time!
```

**Fix:** Use logical clocks (Lamport clocks, vector clocks) or synchronized clocks (like Google's TrueTime).

### Performance Gotchas

#### Strong Consistency Latency

```
Single datacenter:
  - Consensus round-trip: 1-5ms
  - Total write latency: 5-20ms

Multi-datacenter (US East to US West):
  - Network round-trip: 60-80ms
  - Consensus: 2-3 round-trips
  - Total write latency: 150-250ms

Global (US to Europe):
  - Network round-trip: 100-150ms
  - Total write latency: 300-500ms
```

Strong consistency across continents is SLOW.

#### Eventual Consistency Complexity

```
With eventual consistency, you need to handle:
1. Stale reads (show outdated data)
2. Conflicts (two writes to same key)
3. Ordering (events arrive out of order)
4. Retries (messages delivered multiple times)

Each requires careful design and testing.
```

---

## 8️⃣ When NOT to Use This

### When NOT to Use Strong Consistency

1. **High-throughput, low-latency requirements**
   - Gaming leaderboards
   - Social media feeds
   - Recommendation systems

2. **Global distribution with latency requirements**
   - If you need <100ms response time globally
   - Strong consistency adds 100-300ms for cross-continent coordination

3. **When availability is more important than consistency**
   - Shopping carts (better to have duplicates than be unavailable)
   - View counters (approximate is fine)

### When NOT to Use Eventual Consistency

1. **Financial transactions**
   - Bank transfers
   - Payment processing
   - Inventory management (for purchase)

2. **Unique constraints**
   - Username registration
   - Ticket booking (limited inventory)
   - Auction bidding

3. **When users expect immediate feedback**
   - "Your post was published" but they can't see it
   - Confusing and frustrating UX

---

## 9️⃣ Comparison with Alternatives

### Consistency Model Comparison

| Model | Latency | Availability | Complexity | Use Case |
|-------|---------|--------------|------------|----------|
| Linearizable | High | Lower | Low (simple to reason) | Financial, inventory |
| Sequential | Medium | Medium | Medium | Distributed locks |
| Causal | Low-Medium | High | High | Collaborative editing |
| Eventual | Low | Highest | High (conflict resolution) | Social media, caching |
| Read-your-writes | Low | High | Medium | User-facing apps |

### Database Consistency Options

| Database | Default | Options |
|----------|---------|---------|
| PostgreSQL | Strong (single node) | Sync/async replication |
| MySQL | Strong (single node) | Semi-sync replication |
| MongoDB | Eventual | Configurable read/write concern |
| Cassandra | Eventual | Tunable consistency levels |
| DynamoDB | Eventual | Strong read option |
| Spanner | Strong | Always strong |
| CockroachDB | Strong | Serializable isolation |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q1: What's the difference between strong and eventual consistency?**

**Answer:**
Strong consistency guarantees that after a write completes, all subsequent reads will return that value. It's like a single-server database, everyone sees the same data.

Eventual consistency only guarantees that if you stop writing, eventually all replicas will have the same data. In the meantime, different readers might see different values.

Strong consistency is easier to program against but slower and less available. Eventual consistency is faster and more available but requires handling stale reads and conflicts.

**Q2: Why would anyone choose eventual consistency?**

**Answer:**
Three main reasons:

1. **Performance**: No coordination means writes complete in milliseconds instead of tens or hundreds of milliseconds.

2. **Availability**: If you can't reach a quorum of nodes, strong consistency fails. Eventual consistency can continue operating with any single node.

3. **Cost**: Strong consistency requires more network round-trips and more powerful hardware to handle the coordination overhead.

For many use cases (social media feeds, view counters, recommendations), slightly stale data is acceptable and the performance/availability benefits are worth it.

### L5 (Senior) Questions

**Q3: How would you implement read-your-writes consistency in a globally distributed system?**

**Answer:**
Several approaches:

1. **Sticky sessions**: Route all requests from a user to the same datacenter/replica. The replica they write to is the replica they read from.

2. **Version tracking**: After a write, give the client a token containing the write timestamp. On reads, the client sends this token, and the server ensures it reads from a replica that has at least that version.

3. **Synchronous replication for user's region**: When a user writes, synchronously replicate to their local datacenter, but asynchronously to others. Reads go to the local datacenter.

4. **Write-through cache**: Cache the user's recent writes on the edge. Serve reads from cache if present, otherwise from replica.

The right choice depends on your latency requirements, infrastructure, and consistency needs for other users.

**Q4: How do you handle conflicts in an eventually consistent system?**

**Answer:**
Several strategies:

1. **Last-Write-Wins (LWW)**: Use timestamps, keep the latest. Simple but can lose data.

2. **First-Write-Wins**: Keep the first value, reject later writes. Good for immutable data.

3. **Merge**: Application-specific logic to combine conflicting values. Example: for a set, union the elements.

4. **CRDTs**: Use data structures designed to merge automatically without conflicts. Examples: G-Counter, OR-Set, LWW-Register.

5. **Application-level resolution**: Store all conflicting versions, let the application or user resolve. Amazon's shopping cart does this.

The choice depends on your data semantics. Counters use CRDTs. Documents might use operational transformation. Financial data shouldn't have conflicts (use strong consistency).

### L6 (Staff) Questions

**Q5: Design a globally distributed counter that's both fast and accurate.**

**Answer:**
This is a classic tradeoff problem. Here's a tiered approach:

**Layer 1: Local counters (eventual consistency)**
- Each datacenter maintains a local count
- Increments are instant (no coordination)
- Local reads are fast but approximate

**Layer 2: Regional aggregation**
- Every few seconds, local counters report to regional aggregators
- Regional aggregators sum the counts

**Layer 3: Global aggregation**
- Regional totals are periodically sent to a global coordinator
- The global count is "eventually accurate"

**For display**: Show the regional count (fast, slightly stale)
**For billing/analytics**: Use the global count (slower, accurate)

**For exact counts** (like inventory):
- Use a CRDT (G-Counter) that can be merged without conflicts
- Or accept higher latency and use strong consistency

**Q6: How does Google Spanner achieve global strong consistency?**

**Answer:**
Spanner uses three key innovations:

1. **TrueTime API**: GPS receivers and atomic clocks in every datacenter provide globally synchronized time with bounded uncertainty (typically 1-7ms).

2. **Commit-wait**: After a transaction commits, Spanner waits for the TrueTime uncertainty interval to pass before acknowledging. This ensures that any subsequent transaction will have a higher timestamp.

3. **Paxos groups**: Data is replicated across datacenters using Paxos consensus. Writes require a majority of replicas to agree.

The result is linearizable transactions globally, but with latency that includes:
- Cross-datacenter Paxos round-trips (100-200ms)
- TrueTime commit-wait (1-7ms)

Spanner trades latency for consistency. It's appropriate for financial systems, not social media feeds.

---

## 1️⃣1️⃣ One Clean Mental Summary

Consistency models define what values reads can return given the writes that have happened. Strong consistency (linearizability) means everyone sees the same data in real-time order, but it's slow and reduces availability. Eventual consistency means replicas will converge "eventually," but readers might see stale data in the meantime, it's fast and highly available but requires handling conflicts. Most production systems use a mix: strong consistency for critical operations (payments, inventory), eventual consistency for high-volume, latency-sensitive operations (feeds, counters), and read-your-writes for user-facing features (so users see their own changes immediately). The key skill is matching the consistency model to the business requirement, not every piece of data needs the same guarantees.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│              CONSISTENCY MODELS CHEAT SHEET                  │
├─────────────────────────────────────────────────────────────┤
│ STRONG (Linearizable)                                        │
│   ✓ All reads see latest write                              │
│   ✓ Easy to reason about                                    │
│   ✗ High latency (coordination required)                    │
│   ✗ Lower availability during partitions                    │
│   → Use for: payments, inventory, unique constraints        │
├─────────────────────────────────────────────────────────────┤
│ EVENTUAL                                                     │
│   ✓ Low latency (no coordination)                           │
│   ✓ High availability                                       │
│   ✗ Stale reads possible                                    │
│   ✗ Conflicts need resolution                               │
│   → Use for: feeds, counters, caches, recommendations       │
├─────────────────────────────────────────────────────────────┤
│ CAUSAL                                                       │
│   ✓ Related operations ordered correctly                    │
│   ✓ Better than eventual, cheaper than strong               │
│   ✗ Complex to implement                                    │
│   → Use for: collaborative editing, chat                    │
├─────────────────────────────────────────────────────────────┤
│ READ-YOUR-WRITES                                             │
│   ✓ Users see their own changes                             │
│   ✓ Good UX without full strong consistency                 │
│   → Use for: any user-facing write operation                │
├─────────────────────────────────────────────────────────────┤
│ CONFLICT RESOLUTION                                          │
│   LWW: Last-write-wins (simple, may lose data)              │
│   CRDT: Automatic merge (complex, no data loss)             │
│   App-level: Store conflicts, let app resolve               │
└─────────────────────────────────────────────────────────────┘
```

