# Time & Clocks in Distributed Systems

## 0️⃣ Prerequisites

Before diving into distributed time and clocks, you should understand:

- **Distributed Systems** (covered in Phase 1, Topic 01): Systems where multiple computers communicate over unreliable networks
- **Consistency Models** (covered in Phase 1, Topic 12): How different systems guarantee data consistency
- **Failure Modes** (covered in Phase 1, Topic 07): Network partitions, node failures, and message loss

**Quick refresher on distributed systems**: In a distributed system, you have multiple computers (nodes) that need to work together. They communicate by sending messages over a network. The network can delay, lose, or reorder messages. Nodes can fail at any time. There is no shared memory or global state that all nodes can access simultaneously.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

Imagine you're building a distributed database. Two users update the same record at almost the same time:

```
User A (in New York): UPDATE balance SET amount = 100 at 10:00:00.000
User B (in London):   UPDATE balance SET amount = 200 at 10:00:00.000
```

Which update should win? You might say "the one that happened first." But here's the problem: **how do you know which one happened first?**

- Server A's clock says User A's request arrived at 10:00:00.000
- Server B's clock says User B's request arrived at 10:00:00.000
- But Server A's clock is 50ms ahead of Server B's clock
- So did User A's request actually happen before or after User B's?

This is the **distributed time problem**: there is no global clock that all nodes agree on.

### What Systems Looked Like Before This Was Solved

In early distributed systems:

1. **Lost updates**: When two updates happened "at the same time," one would silently disappear
2. **Inconsistent ordering**: Different nodes would see events in different orders
3. **Phantom causality violations**: A reply could appear before the message it was replying to
4. **Debugging nightmares**: Logs from different servers couldn't be correlated because timestamps were meaningless across machines

### What Breaks Without Proper Time Handling

**Without distributed clocks:**

- **Databases**: Last-write-wins becomes arbitrary, not based on actual order
- **Event sourcing**: Events get replayed in wrong order, corrupting state
- **Distributed transactions**: Cannot determine if transaction A committed before B
- **Conflict resolution**: No way to decide which concurrent update should win
- **Debugging**: Cannot reconstruct what happened across multiple services
- **Caching**: Cache invalidation timing becomes unreliable

### Real Examples of the Problem

**Amazon's Dynamo**:
Amazon discovered that using wall-clock time for conflict resolution in their shopping cart system led to unpredictable behavior. A customer might add an item to their cart, but if the replica they wrote to had a slow clock, another replica's "older" version could overwrite their addition. This led to the development of vector clocks in Dynamo.

**Google Spanner**:
Google needed globally consistent transactions for their advertising system. They couldn't rely on regular clocks because even NTP (Network Time Protocol) has milliseconds of uncertainty. They built TrueTime, a system using GPS and atomic clocks that provides bounded time uncertainty, enabling them to order transactions globally.

**The 2012 Leap Second Bug**:
When a leap second was added on June 30, 2012, many Linux systems experienced high CPU usage and crashes. The kernel's handling of the extra second caused timing loops. Reddit, Mozilla, Gawker, and many other sites went down. This showed how sensitive distributed systems are to time anomalies.

---

## 2️⃣ Intuition and Mental Model

### The Orchestra Analogy

Imagine an orchestra performing a symphony. Every musician needs to play their notes at the right time relative to other musicians.

**Scenario 1: Single Conductor (Physical Clock with Perfect Sync)**
- One conductor waves a baton
- Everyone watches the same conductor
- Perfect synchronization
- But what if some musicians are far away and see the baton movement with a delay?

**Scenario 2: No Conductor, Just Sheet Music (Logical Clocks)**
- No central timekeeper
- Musicians just follow the order of notes in the sheet music
- "After the violins finish measure 4, I start measure 5"
- They don't need to know the exact time, just the order

**Scenario 3: Section Leaders (Vector Clocks)**
- Each section (strings, brass, woodwinds) has a leader
- Leaders coordinate with each other
- Each section tracks what other sections have done
- "Brass is on measure 6, strings are on measure 5, I should wait for strings"

This analogy captures the key insight: **in distributed systems, we often care more about order than about exact time**.

### The Three Types of Time

```
┌─────────────────────────────────────────────────────────────┐
│                    TYPES OF TIME                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. PHYSICAL TIME (Wall Clock)                              │
│     "What time is it right now?"                            │
│     - Measured by clocks (quartz, atomic, GPS)              │
│     - Subject to drift, leap seconds, NTP corrections       │
│     - Can go backwards! (NTP adjustment)                    │
│                                                              │
│  2. LOGICAL TIME (Lamport Clock)                            │
│     "What is the order of events?"                          │
│     - Just a counter that increments                        │
│     - Only tells you order, not duration                    │
│     - Never goes backwards                                  │
│                                                              │
│  3. HYBRID TIME (HLC, TrueTime)                             │
│     "Best of both worlds"                                   │
│     - Physical time for approximate ordering                │
│     - Logical component for tie-breaking                    │
│     - Bounded uncertainty                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ How It Works Internally

### Physical Clocks: The Problem

Every computer has a physical clock, typically a quartz crystal oscillator. The problem is: **clocks drift**.

**Clock Drift Explained:**

```
Ideal clock:  |----|----|----|----|----|----|----| (exactly 1 second each)
Real clock:   |----|----|------|----|----|--|-----| (varies slightly)

Drift rate: typically 10-200 parts per million (ppm)
- 100 ppm = 8.64 seconds per day
- A "good" server clock drifts ~50ms per hour
```

**What causes drift:**
- Temperature changes (quartz crystals are temperature-sensitive)
- Aging of the crystal
- Power supply variations
- Manufacturing variations

**NTP (Network Time Protocol):**

NTP synchronizes clocks across the internet. It works like this:

```
Client                                          NTP Server
   │                                                │
   │  1. Request (T1 = client send time)           │
   │ ─────────────────────────────────────────────>│
   │                                                │
   │                    2. Server receives at T2   │
   │                    3. Server responds at T3   │
   │                                                │
   │  4. Response (T4 = client receive time)       │
   │ <─────────────────────────────────────────────│
   │                                                │
   │  Calculate:                                   │
   │  Round-trip delay = (T4 - T1) - (T3 - T2)    │
   │  Clock offset = ((T2 - T1) + (T3 - T4)) / 2  │
```

**NTP Limitations:**
- Accuracy: 1-50ms over the internet, 0.1-1ms on LAN
- Asymmetric network delays cause errors
- Clock can jump backwards (dangerous for distributed systems!)

### Logical Clocks: Lamport Timestamps

Leslie Lamport (Turing Award winner) invented logical clocks in 1978. The key insight: **we don't need to know the actual time, just the order of events**.

**The Rules:**

```
1. Each process maintains a counter C
2. Before executing an event, increment C: C = C + 1
3. When sending a message, include the current C value
4. When receiving a message with timestamp T:
   C = max(C, T) + 1
```

**Visual Example:**

```
Process A        Process B        Process C
    │                │                │
    │ C=1            │                │
    ├───────────────>│                │
    │           C=2 (max(0,1)+1)      │
    │                │                │
    │                │ C=3            │
    │                ├───────────────>│
    │                │           C=4 (max(0,3)+1)
    │                │                │
    │ C=2            │                │
    │<───────────────┼────────────────┤
    │ C=5 (max(2,4)+1)                │
```

**What Lamport Clocks Guarantee:**

If event A "happened before" event B (written A → B), then C(A) < C(B).

**What They DON'T Guarantee:**

If C(A) < C(B), we CANNOT conclude that A → B. The events might be concurrent!

```
Process A: C=5 (some local event)
Process B: C=3 (some local event)

C(B) < C(A), but B didn't happen before A.
They're concurrent (neither caused the other).
```

### Vector Clocks: Detecting Concurrency

Vector clocks extend Lamport clocks to detect concurrent events.

**The Idea:**
Instead of a single counter, each process maintains a vector of counters, one for each process in the system.

```
Process A's vector: [A's count, B's count, C's count]
Process B's vector: [A's count, B's count, C's count]
Process C's vector: [A's count, B's count, C's count]
```

**The Rules:**

```
1. Each process i maintains vector V[i]
2. Before executing an event: V[i][i] = V[i][i] + 1
3. When sending a message, include the entire vector V[i]
4. When receiving a message with vector V':
   For each j: V[i][j] = max(V[i][j], V'[j])
   Then: V[i][i] = V[i][i] + 1
```

**Visual Example:**

```
Process A              Process B              Process C
V=[0,0,0]             V=[0,0,0]             V=[0,0,0]
    │                      │                      │
    │ V=[1,0,0]           │                      │
    ├─────────────────────>│                      │
    │                 V=[1,1,0]                   │
    │                      │                      │
    │                      │ V=[1,2,0]            │
    │                      ├─────────────────────>│
    │                      │                 V=[1,2,1]
    │                      │                      │
    │ V=[2,0,0]           │                      │
    │                      │                      │
```

**Comparing Vector Clocks:**

```
V1 = [2,3,1]
V2 = [1,4,1]

V1 < V2?  Check if all V1[i] <= V2[i] and at least one V1[i] < V2[i]
  V1[0]=2 > V2[0]=1  ✗
  
V2 < V1?  
  V2[1]=4 > V1[1]=3  ✗

Neither is less than the other → CONCURRENT events!
```

**Java Implementation:**

```java
package com.systemdesign.time;

import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;

/**
 * Vector clock implementation for tracking causality in distributed systems.
 * Each process maintains a vector of counters, one per process.
 */
public class VectorClock {
    
    private final String processId;
    private final Map<String, Long> clock;
    
    public VectorClock(String processId) {
        this.processId = processId;
        this.clock = new HashMap<>();
        this.clock.put(processId, 0L);
    }
    
    /**
     * Increment own counter before a local event or sending a message.
     */
    public void tick() {
        clock.merge(processId, 1L, Long::sum);
    }
    
    /**
     * Update clock when receiving a message.
     * Takes element-wise maximum, then increments own counter.
     */
    public void update(VectorClock other) {
        // Merge: take max of each component
        for (Map.Entry<String, Long> entry : other.clock.entrySet()) {
            clock.merge(entry.getKey(), entry.getValue(), Math::max);
        }
        // Increment own counter
        tick();
    }
    
    /**
     * Get a copy of this clock to send with a message.
     */
    public VectorClock copy() {
        VectorClock copy = new VectorClock(this.processId);
        copy.clock.putAll(this.clock);
        return copy;
    }
    
    /**
     * Compare two vector clocks.
     * Returns:
     *   -1 if this happened before other
     *    1 if other happened before this
     *    0 if concurrent (neither happened before the other)
     */
    public int compareTo(VectorClock other) {
        boolean thisLessOrEqual = true;
        boolean otherLessOrEqual = true;
        boolean thisStrictlyLess = false;
        boolean otherStrictlyLess = false;
        
        // Get all process IDs from both clocks
        java.util.Set<String> allProcesses = new java.util.HashSet<>();
        allProcesses.addAll(this.clock.keySet());
        allProcesses.addAll(other.clock.keySet());
        
        for (String proc : allProcesses) {
            long thisVal = this.clock.getOrDefault(proc, 0L);
            long otherVal = other.clock.getOrDefault(proc, 0L);
            
            if (thisVal > otherVal) {
                otherLessOrEqual = false;
                thisStrictlyLess = false;
            }
            if (otherVal > thisVal) {
                thisLessOrEqual = false;
                otherStrictlyLess = false;
            }
            if (thisVal < otherVal) {
                thisStrictlyLess = true;
            }
            if (otherVal < thisVal) {
                otherStrictlyLess = true;
            }
        }
        
        if (thisLessOrEqual && thisStrictlyLess) {
            return -1; // this happened before other
        }
        if (otherLessOrEqual && otherStrictlyLess) {
            return 1;  // other happened before this
        }
        return 0; // concurrent
    }
    
    /**
     * Check if this clock happened before another.
     */
    public boolean happenedBefore(VectorClock other) {
        return compareTo(other) == -1;
    }
    
    /**
     * Check if two clocks represent concurrent events.
     */
    public boolean isConcurrentWith(VectorClock other) {
        return compareTo(other) == 0;
    }
    
    @Override
    public String toString() {
        return clock.toString();
    }
}
```

### Hybrid Logical Clocks (HLC)

Vector clocks have a problem: they grow with the number of processes. In a system with millions of clients, this is impractical.

Hybrid Logical Clocks (HLC) combine physical and logical time:

```
HLC = (physical_time, logical_counter)

- physical_time: wall clock time, provides approximate ordering
- logical_counter: ensures causality when physical times are equal
```

**The Rules:**

```
When a local event happens or sending a message:
  l' = max(l, physical_time)
  if l' == l:
    c = c + 1
  else:
    c = 0
  l = l'

When receiving a message with timestamp (l', c'):
  l'' = max(l, l', physical_time)
  if l'' == l == l':
    c = max(c, c') + 1
  else if l'' == l:
    c = c + 1
  else if l'' == l':
    c = c' + 1
  else:
    c = 0
  l = l''
```

**Java Implementation:**

```java
package com.systemdesign.time;

/**
 * Hybrid Logical Clock implementation.
 * Combines physical time with a logical counter for bounded-size timestamps
 * that still capture causality.
 */
public class HybridLogicalClock {
    
    private long logicalTime;    // Physical time component (milliseconds)
    private int counter;         // Logical counter for tie-breaking
    
    public HybridLogicalClock() {
        this.logicalTime = System.currentTimeMillis();
        this.counter = 0;
    }
    
    /**
     * Generate a timestamp for a local event or outgoing message.
     */
    public synchronized HLCTimestamp now() {
        long physicalTime = System.currentTimeMillis();
        
        if (physicalTime > logicalTime) {
            // Physical time has advanced, reset counter
            logicalTime = physicalTime;
            counter = 0;
        } else {
            // Physical time hasn't advanced, increment counter
            counter++;
        }
        
        return new HLCTimestamp(logicalTime, counter);
    }
    
    /**
     * Update clock when receiving a message with a timestamp.
     */
    public synchronized HLCTimestamp receive(HLCTimestamp incoming) {
        long physicalTime = System.currentTimeMillis();
        
        if (physicalTime > logicalTime && physicalTime > incoming.logicalTime) {
            // Physical time is ahead of both
            logicalTime = physicalTime;
            counter = 0;
        } else if (logicalTime > incoming.logicalTime) {
            // Our logical time is ahead
            counter++;
        } else if (incoming.logicalTime > logicalTime) {
            // Incoming logical time is ahead
            logicalTime = incoming.logicalTime;
            counter = incoming.counter + 1;
        } else {
            // Logical times are equal
            counter = Math.max(counter, incoming.counter) + 1;
        }
        
        return new HLCTimestamp(logicalTime, counter);
    }
    
    /**
     * Immutable timestamp value.
     */
    public record HLCTimestamp(long logicalTime, int counter) implements Comparable<HLCTimestamp> {
        
        @Override
        public int compareTo(HLCTimestamp other) {
            int timeCompare = Long.compare(this.logicalTime, other.logicalTime);
            if (timeCompare != 0) {
                return timeCompare;
            }
            return Integer.compare(this.counter, other.counter);
        }
        
        @Override
        public String toString() {
            return logicalTime + "." + counter;
        }
    }
}
```

### Google's TrueTime

TrueTime is Google's solution for Spanner. Instead of pretending clocks are synchronized, it acknowledges uncertainty.

**The API:**

```
TrueTime.now() returns an interval [earliest, latest]
- The actual time is guaranteed to be within this interval
- Typical uncertainty: 1-7 milliseconds
```

**How it works:**

```
┌─────────────────────────────────────────────────────────────┐
│                    TRUETIME ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Each datacenter has:                                       │
│  ┌─────────────┐  ┌─────────────┐                          │
│  │ GPS Receiver│  │Atomic Clock │                          │
│  └──────┬──────┘  └──────┬──────┘                          │
│         │                 │                                 │
│         └────────┬────────┘                                │
│                  ▼                                          │
│         ┌───────────────┐                                  │
│         │ Time Master   │                                  │
│         └───────┬───────┘                                  │
│                 │                                          │
│    ┌────────────┼────────────┐                             │
│    ▼            ▼            ▼                             │
│ ┌──────┐   ┌──────┐   ┌──────┐                            │
│ │Server│   │Server│   │Server│                            │
│ └──────┘   └──────┘   └──────┘                            │
│                                                              │
│  GPS provides absolute time (within ~1μs)                   │
│  Atomic clock provides stable time between GPS updates      │
│  Uncertainty comes from network delays to time master       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Commit-Wait Protocol:**

```
To ensure linearizability, Spanner waits for uncertainty to pass:

1. Transaction commits at time T
2. Spanner waits until TrueTime.now().earliest > T
3. Only then is the commit acknowledged

This ensures any subsequent transaction will have a timestamp > T
```

---

## 4️⃣ Simulation-First Explanation

Let's trace through a concrete scenario to understand how different clocks behave.

### Scenario: Distributed Key-Value Store

**Setup:**
- Three nodes: A, B, C
- User writes key "x" to node A
- User reads key "x" from node B
- We need to ensure B returns the correct value

### With Physical Clocks Only (Broken)

```
Time (real):     0ms        50ms       100ms      150ms
                  │          │          │          │
Node A clock:    0ms        52ms       102ms      154ms  (2ms fast)
Node B clock:    0ms        48ms        98ms      148ms  (2ms slow)

User writes to A at real time 50ms:
  A's clock says: 52ms
  A stores: {key: "x", value: "hello", timestamp: 52}
  A replicates to B (arrives at real time 100ms)
  
User reads from B at real time 75ms:
  B's clock says: 73ms
  B hasn't received the replication yet!
  B returns: null (or old value)
  
Problem: User wrote at 50ms, read at 75ms, but didn't see their write!
```

### With Lamport Clocks

```
Initial state:
  A.clock = 0
  B.clock = 0
  C.clock = 0

Step 1: User writes to A
  A.clock = 1
  A stores: {key: "x", value: "hello", lamport: 1}
  A sends to B: {key: "x", value: "hello", lamport: 1}

Step 2: B receives replication
  B.clock = max(0, 1) + 1 = 2
  B stores: {key: "x", value: "hello", lamport: 1}

Step 3: User reads from B
  B.clock = 3
  B returns: {key: "x", value: "hello", lamport: 1}

Now user knows: the write (lamport=1) happened before the read (lamport=3)
```

### With Vector Clocks (Detecting Conflicts)

```
Initial state:
  A.vector = [0, 0, 0]
  B.vector = [0, 0, 0]

Scenario: Two concurrent writes to same key

Step 1: User 1 writes to A
  A.vector = [1, 0, 0]
  A stores: {key: "x", value: "hello", vector: [1,0,0]}

Step 2: User 2 writes to B (before A's update arrives)
  B.vector = [0, 1, 0]
  B stores: {key: "x", value: "world", vector: [0,1,0]}

Step 3: A and B exchange updates
  A receives B's update:
    Compare [1,0,0] vs [0,1,0]
    [1,0,0] is NOT < [0,1,0] (1 > 0)
    [0,1,0] is NOT < [1,0,0] (1 > 0)
    CONFLICT DETECTED!
  
  A now has two versions:
    {key: "x", value: "hello", vector: [1,0,0]}
    {key: "x", value: "world", vector: [0,1,0]}
  
  Application must resolve: merge, pick one, or ask user
```

### With HLC (Practical Production Use)

```
Initial state:
  A.hlc = (1000, 0)  // physical time 1000ms, counter 0
  B.hlc = (1000, 0)

Step 1: User writes to A at physical time 1050ms
  A.hlc = (1050, 0)  // physical time advanced, reset counter
  A stores: {key: "x", value: "hello", hlc: (1050, 0)}
  A sends to B

Step 2: Event at A at physical time 1050ms (same millisecond)
  A.hlc = (1050, 1)  // same physical time, increment counter
  
Step 3: B receives at physical time 1055ms
  B receives message with hlc (1050, 0)
  B's physical time (1055) > message logical time (1050)
  B.hlc = (1055, 0)  // use physical time, reset counter
  B stores: {key: "x", value: "hello", hlc: (1050, 0)}

Ordering is preserved: (1050, 0) < (1055, 0)
```

---

## 5️⃣ How Engineers Actually Use This in Production

### At Major Companies

**Google (Spanner)**:
- Uses TrueTime for global transaction ordering
- GPS + atomic clocks in every datacenter
- Commit-wait ensures linearizability
- Typical uncertainty: 1-7ms
- Used for AdWords, Google Play, and other critical systems

**Amazon (DynamoDB)**:
- Uses vector clocks (simplified) for conflict detection
- Last-write-wins based on wall clock as default
- Application can choose to see all conflicting versions
- Suitable for shopping carts, session stores

**CockroachDB**:
- Uses HLC (Hybrid Logical Clocks)
- Provides serializable transactions without TrueTime
- Trades off some latency for correctness
- Open-source alternative to Spanner

**Apache Cassandra**:
- Uses wall-clock timestamps with last-write-wins
- Tunable consistency (quorum reads/writes)
- Accepts that clocks may be skewed
- Suitable for time-series data, logs

**MongoDB**:
- Uses a form of logical clocks for replication
- Oplog (operation log) entries have timestamps
- Causal consistency sessions available

### Choosing the Right Clock

```
┌─────────────────────────────────────────────────────────────┐
│                 CLOCK TYPE DECISION TREE                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Do you need to order events across nodes?                  │
│  ├── NO → Physical clock is fine (logs, metrics)            │
│  │                                                          │
│  └── YES → Do you need to detect concurrent events?         │
│            ├── NO → Lamport clock (simple ordering)         │
│            │                                                │
│            └── YES → How many processes?                    │
│                      ├── Small (< 100) → Vector clock       │
│                      │                                      │
│                      └── Large → HLC or TrueTime            │
│                                                             │
│  Need global wall-clock ordering?                           │
│  ├── YES, and have budget → TrueTime (GPS + atomic)        │
│  └── YES, but no budget → HLC with NTP                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Production Configuration

**NTP Configuration (Linux):**

```bash
# /etc/ntp.conf

# Use multiple time sources for accuracy
server 0.pool.ntp.org iburst
server 1.pool.ntp.org iburst
server 2.pool.ntp.org iburst
server 3.pool.ntp.org iburst

# Limit clock adjustments (prevent large jumps)
tinker panic 0  # Don't panic on large offsets
tinker step 0.1 # Step if offset > 100ms

# Log statistics
statsdir /var/log/ntpstats/
statistics loopstats peerstats clockstats
```

**Chrony (Modern Alternative to NTP):**

```bash
# /etc/chrony/chrony.conf

# Use cloud provider's time service
server 169.254.169.123 prefer iburst  # AWS Time Sync
server time.google.com iburst         # Google Public NTP

# Allow large initial adjustment
makestep 1.0 3

# Enable hardware timestamping if available
hwtimestamp *
```

---

## 6️⃣ How to Implement or Apply It

### Complete Java Implementation: Distributed Event Ordering

Let's build a practical system that uses HLC for event ordering.

#### Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```

#### Lamport Clock Implementation

```java
package com.systemdesign.time;

import java.util.concurrent.atomic.AtomicLong;

/**
 * Thread-safe Lamport clock implementation.
 * Provides logical timestamps for ordering events in distributed systems.
 */
public class LamportClock {
    
    private final AtomicLong counter;
    
    public LamportClock() {
        this.counter = new AtomicLong(0);
    }
    
    /**
     * Generate a timestamp for a local event.
     * Must be called before sending a message or recording an event.
     */
    public long tick() {
        return counter.incrementAndGet();
    }
    
    /**
     * Update the clock when receiving a message.
     * Takes the maximum of local and received timestamp, then increments.
     * 
     * @param receivedTimestamp The timestamp from the received message
     * @return The new local timestamp
     */
    public long receive(long receivedTimestamp) {
        long current;
        long newValue;
        do {
            current = counter.get();
            // Take max of current and received, then add 1
            newValue = Math.max(current, receivedTimestamp) + 1;
        } while (!counter.compareAndSet(current, newValue));
        return newValue;
    }
    
    /**
     * Get current timestamp without incrementing.
     * Useful for reading the current logical time.
     */
    public long current() {
        return counter.get();
    }
}
```

#### Event Store with HLC

```java
package com.systemdesign.time;

import java.util.*;
import java.util.concurrent.*;

/**
 * An event store that uses Hybrid Logical Clocks for ordering.
 * Events are stored in causal order, not wall-clock order.
 */
public class HLCEventStore {
    
    private final HybridLogicalClock hlc;
    private final ConcurrentSkipListMap<HybridLogicalClock.HLCTimestamp, Event> events;
    private final String nodeId;
    
    public HLCEventStore(String nodeId) {
        this.nodeId = nodeId;
        this.hlc = new HybridLogicalClock();
        this.events = new ConcurrentSkipListMap<>();
    }
    
    /**
     * Record a local event.
     */
    public Event recordEvent(String type, Map<String, Object> data) {
        HybridLogicalClock.HLCTimestamp timestamp = hlc.now();
        Event event = new Event(
            UUID.randomUUID().toString(),
            nodeId,
            type,
            data,
            timestamp
        );
        events.put(timestamp, event);
        return event;
    }
    
    /**
     * Receive an event from another node.
     * Updates local clock and stores the event.
     */
    public void receiveEvent(Event event) {
        // Update our clock based on the received timestamp
        hlc.receive(event.timestamp());
        // Store the event with its original timestamp
        events.put(event.timestamp(), event);
    }
    
    /**
     * Get all events in causal order.
     */
    public List<Event> getEventsInOrder() {
        return new ArrayList<>(events.values());
    }
    
    /**
     * Get events after a specific timestamp.
     */
    public List<Event> getEventsAfter(HybridLogicalClock.HLCTimestamp after) {
        return new ArrayList<>(events.tailMap(after, false).values());
    }
    
    /**
     * Get current HLC timestamp (for sending with messages).
     */
    public HybridLogicalClock.HLCTimestamp getCurrentTimestamp() {
        return hlc.now();
    }
    
    public record Event(
        String id,
        String sourceNode,
        String type,
        Map<String, Object> data,
        HybridLogicalClock.HLCTimestamp timestamp
    ) {}
}
```

#### Distributed Key-Value Store with Conflict Detection

```java
package com.systemdesign.time;

import java.util.*;
import java.util.concurrent.*;

/**
 * A key-value store that uses vector clocks for conflict detection.
 * When concurrent writes occur, both versions are kept for resolution.
 */
public class VectorClockKVStore {
    
    private final String nodeId;
    private final Map<String, List<VersionedValue>> store;
    private final VectorClock clock;
    
    public VectorClockKVStore(String nodeId) {
        this.nodeId = nodeId;
        this.store = new ConcurrentHashMap<>();
        this.clock = new VectorClock(nodeId);
    }
    
    /**
     * Put a value. If there are existing values, check for conflicts.
     */
    public PutResult put(String key, String value, VectorClock clientClock) {
        clock.tick();
        
        VectorClock writeVersion;
        if (clientClock != null) {
            // Client is updating based on a previous read
            clock.update(clientClock);
            writeVersion = clock.copy();
        } else {
            // Fresh write
            writeVersion = clock.copy();
        }
        
        VersionedValue newValue = new VersionedValue(value, writeVersion);
        
        List<VersionedValue> existing = store.get(key);
        if (existing == null || existing.isEmpty()) {
            // No existing values, simple case
            store.put(key, new CopyOnWriteArrayList<>(List.of(newValue)));
            return new PutResult(false, List.of(newValue));
        }
        
        // Check if new value supersedes existing values
        List<VersionedValue> newVersions = new ArrayList<>();
        boolean hasConflict = false;
        
        for (VersionedValue ev : existing) {
            int comparison = writeVersion.compareTo(ev.version());
            if (comparison == 1) {
                // New value is strictly after existing, existing is obsolete
                continue;
            } else if (comparison == -1) {
                // Existing value is strictly after new, new is obsolete
                return new PutResult(false, existing);
            } else {
                // Concurrent! Keep both
                newVersions.add(ev);
                hasConflict = true;
            }
        }
        
        newVersions.add(newValue);
        store.put(key, new CopyOnWriteArrayList<>(newVersions));
        
        return new PutResult(hasConflict, newVersions);
    }
    
    /**
     * Get a value. May return multiple versions if there are conflicts.
     */
    public GetResult get(String key) {
        List<VersionedValue> values = store.get(key);
        if (values == null || values.isEmpty()) {
            return new GetResult(null, false, clock.copy());
        }
        
        if (values.size() == 1) {
            return new GetResult(values.get(0), false, clock.copy());
        }
        
        // Multiple versions = conflict
        return new GetResult(values, true, clock.copy());
    }
    
    /**
     * Resolve a conflict by picking a winner.
     * The winning version's clock becomes the new version.
     */
    public void resolveConflict(String key, VersionedValue winner) {
        clock.update(winner.version());
        clock.tick();
        
        VersionedValue resolved = new VersionedValue(winner.value(), clock.copy());
        store.put(key, new CopyOnWriteArrayList<>(List.of(resolved)));
    }
    
    /**
     * Receive a replicated value from another node.
     */
    public void receiveReplication(String key, VersionedValue incoming) {
        clock.update(incoming.version());
        
        List<VersionedValue> existing = store.computeIfAbsent(key, 
            k -> new CopyOnWriteArrayList<>());
        
        // Remove values that are superseded by incoming
        existing.removeIf(ev -> incoming.version().compareTo(ev.version()) == 1);
        
        // Add incoming if it's not superseded by any existing
        boolean superseded = existing.stream()
            .anyMatch(ev -> ev.version().compareTo(incoming.version()) == 1);
        
        if (!superseded) {
            existing.add(incoming);
        }
    }
    
    public record VersionedValue(String value, VectorClock version) {}
    
    public record PutResult(boolean hasConflict, List<VersionedValue> versions) {}
    
    public record GetResult(Object value, boolean hasConflict, VectorClock readVersion) {
        @SuppressWarnings("unchecked")
        public List<VersionedValue> getConflictingVersions() {
            if (value instanceof List) {
                return (List<VersionedValue>) value;
            }
            return value != null ? List.of((VersionedValue) value) : List.of();
        }
    }
}
```

#### Spring Boot Controller

```java
package com.systemdesign.time;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/events")
public class EventController {
    
    private final HLCEventStore eventStore;
    
    public EventController() {
        this.eventStore = new HLCEventStore("node-" + System.getenv("NODE_ID"));
    }
    
    /**
     * Record a new event.
     */
    @PostMapping
    public ResponseEntity<HLCEventStore.Event> createEvent(
            @RequestBody CreateEventRequest request) {
        
        HLCEventStore.Event event = eventStore.recordEvent(
            request.type(),
            request.data()
        );
        
        return ResponseEntity.ok(event);
    }
    
    /**
     * Get all events in causal order.
     */
    @GetMapping
    public ResponseEntity<java.util.List<HLCEventStore.Event>> getEvents(
            @RequestParam(required = false) String afterTimestamp) {
        
        if (afterTimestamp != null) {
            // Parse timestamp and get events after it
            String[] parts = afterTimestamp.split("\\.");
            long logicalTime = Long.parseLong(parts[0]);
            int counter = Integer.parseInt(parts[1]);
            HybridLogicalClock.HLCTimestamp after = 
                new HybridLogicalClock.HLCTimestamp(logicalTime, counter);
            return ResponseEntity.ok(eventStore.getEventsAfter(after));
        }
        
        return ResponseEntity.ok(eventStore.getEventsInOrder());
    }
    
    /**
     * Receive an event from another node (replication).
     */
    @PostMapping("/replicate")
    public ResponseEntity<Void> receiveReplication(
            @RequestBody HLCEventStore.Event event) {
        
        eventStore.receiveEvent(event);
        return ResponseEntity.ok().build();
    }
    
    public record CreateEventRequest(String type, Map<String, Object> data) {}
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

#### 1. Using System.currentTimeMillis() for Ordering

**Wrong:**
```java
// BAD: Wall clock can go backwards, has drift
public void recordEvent(Event event) {
    event.setTimestamp(System.currentTimeMillis());
    events.add(event);
}
```

**Right:**
```java
// GOOD: Use logical or hybrid clock
public void recordEvent(Event event) {
    event.setTimestamp(hlc.now());
    events.add(event);
}
```

#### 2. Ignoring Clock Skew in Distributed Transactions

**Wrong:**
```java
// BAD: Assumes clocks are synchronized
public boolean isTransactionValid(Transaction tx) {
    long now = System.currentTimeMillis();
    return tx.getTimestamp() < now + TIMEOUT;
}
```

**Right:**
```java
// GOOD: Account for clock skew
public boolean isTransactionValid(Transaction tx) {
    long now = System.currentTimeMillis();
    long maxSkew = 5000; // 5 seconds maximum expected skew
    return tx.getTimestamp() < now + TIMEOUT + maxSkew;
}
```

#### 3. Vector Clocks Without Garbage Collection

**Wrong:**
```java
// BAD: Vector clock grows unbounded
public class VectorClock {
    private Map<String, Long> clock = new HashMap<>();
    // Entries are never removed!
}
```

**Right:**
```java
// GOOD: Prune old entries
public class VectorClock {
    private Map<String, Long> clock = new HashMap<>();
    private static final int MAX_ENTRIES = 1000;
    
    public void prune() {
        if (clock.size() > MAX_ENTRIES) {
            // Remove entries with lowest values
            List<Map.Entry<String, Long>> sorted = new ArrayList<>(clock.entrySet());
            sorted.sort(Map.Entry.comparingByValue());
            for (int i = 0; i < sorted.size() - MAX_ENTRIES; i++) {
                clock.remove(sorted.get(i).getKey());
            }
        }
    }
}
```

#### 4. Not Handling NTP Jumps

**Wrong:**
```java
// BAD: Doesn't handle time going backwards
public long getNextSequence() {
    return System.currentTimeMillis();
}
```

**Right:**
```java
// GOOD: Handle backwards jumps
public class MonotonicClock {
    private long lastTime = 0;
    private int counter = 0;
    
    public synchronized long getNextSequence() {
        long now = System.currentTimeMillis();
        if (now <= lastTime) {
            counter++;
            return (lastTime << 16) | counter;
        } else {
            lastTime = now;
            counter = 0;
            return now << 16;
        }
    }
}
```

### Performance Gotchas

#### Lamport Clock Overhead

```
Lamport clock operations are O(1) and very fast.
Overhead: ~10-50 nanoseconds per operation.
Not a concern for most applications.
```

#### Vector Clock Size

```
Vector clock size = O(number of processes)

For 3 nodes: 24 bytes (3 × 8-byte longs)
For 100 nodes: 800 bytes
For 10,000 nodes: 80 KB per message!

Solutions:
1. Use HLC instead (fixed 12 bytes)
2. Prune inactive nodes
3. Use hierarchical clocks
```

#### TrueTime Commit-Wait Latency

```
TrueTime uncertainty: typically 1-7ms
Commit-wait adds this to every transaction.

For write-heavy workloads:
- 1000 writes/second
- 7ms wait each
- 7 seconds of cumulative waiting per second!

Solution: Batch writes when possible
```

---

## 8️⃣ When NOT to Use This

### When Physical Clocks Are Fine

1. **Single-node applications**: No distributed ordering needed
2. **Logs and metrics**: Approximate time is acceptable
3. **Human-readable timestamps**: Users don't notice milliseconds
4. **Batch processing**: Order within batch doesn't matter

### When Lamport Clocks Are Overkill

1. **Strong consistency systems**: Leader handles ordering
2. **Synchronous replication**: Order is implicit in replication
3. **Idempotent operations**: Order doesn't affect outcome

### When Vector Clocks Are Impractical

1. **Large number of clients**: Vector grows too large
2. **High-throughput systems**: Overhead is too high
3. **Systems with strong consistency**: Conflicts can't happen

### Anti-Patterns

1. **Using wall clock for distributed locks**
   - Locks can expire early or late due to clock skew
   - Use fencing tokens instead

2. **Relying on NTP for transaction ordering**
   - NTP has unbounded uncertainty
   - Use logical clocks or TrueTime

3. **Assuming monotonic time**
   - System.currentTimeMillis() can go backwards
   - Use System.nanoTime() for durations (but it's not wall clock)

---

## 9️⃣ Comparison with Alternatives

### Clock Type Comparison

| Aspect | Physical | Lamport | Vector | HLC | TrueTime |
|--------|----------|---------|--------|-----|----------|
| Size | 8 bytes | 8 bytes | 8×N bytes | 12 bytes | 16 bytes |
| Detects causality | No | Partial | Yes | Yes | Yes |
| Detects concurrency | No | No | Yes | No | No |
| Wall-clock correlation | Yes | No | No | Yes | Yes |
| Bounded uncertainty | No | N/A | N/A | No | Yes |
| Cost | Free | Free | Free | Free | $$$ |

### When to Use Each

| Use Case | Recommended Clock |
|----------|-------------------|
| Logging | Physical (NTP synced) |
| Event ordering | Lamport or HLC |
| Conflict detection | Vector clocks |
| Global transactions | TrueTime or HLC |
| Debugging | HLC (has wall-clock component) |

### Database Clock Implementations

| Database | Clock Type | Notes |
|----------|------------|-------|
| Spanner | TrueTime | GPS + atomic clocks |
| CockroachDB | HLC | Software-only |
| Cassandra | Physical | Last-write-wins |
| DynamoDB | Vector (simplified) | For conflict detection |
| MongoDB | Logical | Oplog timestamps |
| YugabyteDB | HLC | Spanner-like without TrueTime |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q1: Why can't we just use wall-clock time to order events in a distributed system?**

**Answer:**
Three main reasons:

1. **Clock drift**: Every computer's clock runs at a slightly different speed. Over time, clocks diverge. A clock might drift 50ms per hour, so after a day, it could be off by over a second.

2. **Clock jumps**: NTP (Network Time Protocol) periodically adjusts clocks. If a clock is ahead, NTP might jump it backwards. This means `timestamp_later < timestamp_earlier`, breaking any ordering based on timestamps.

3. **Network delays**: Even if clocks were perfectly synchronized, network delays are unpredictable. A message sent at time T1 might arrive after a message sent at time T2 if T1's network path is slower.

The solution is to use logical clocks (like Lamport clocks) that only care about order, not actual time.

**Q2: What is a Lamport clock and how does it work?**

**Answer:**
A Lamport clock is a counter that provides logical timestamps for ordering events.

Rules:
1. Each process has a counter, starting at 0
2. Before any event (local or sending a message), increment the counter
3. When sending a message, include the current counter value
4. When receiving a message with timestamp T, set counter = max(counter, T) + 1

This guarantees: if event A causally happened before event B, then timestamp(A) < timestamp(B).

Example:
```
Process A: counter=1 (local event)
Process A: counter=2 (sends message to B with timestamp 2)
Process B: receives message, counter = max(0, 2) + 1 = 3
```

### L5 (Senior) Questions

**Q3: What's the difference between Lamport clocks and vector clocks? When would you use each?**

**Answer:**
**Lamport clocks**:
- Single counter per process
- If A happened before B, then L(A) < L(B)
- BUT: L(A) < L(B) does NOT mean A happened before B
- Cannot detect concurrent events
- O(1) space

**Vector clocks**:
- Array of counters, one per process
- Can determine if A happened before B, B happened before A, or they're concurrent
- Compare element-wise: A < B if all A[i] <= B[i] and at least one A[i] < B[i]
- O(N) space where N is number of processes

**When to use Lamport clocks**:
- Simple ordering is sufficient
- Many processes (vector would be too large)
- Don't need conflict detection

**When to use vector clocks**:
- Need to detect concurrent writes (conflicts)
- Small number of processes (< 100)
- Building eventually consistent systems with conflict resolution

**Q4: How would you implement a distributed timestamp service that provides monotonically increasing timestamps across multiple nodes?**

**Answer:**
Several approaches:

**Approach 1: Centralized timestamp service**
```
- Single node issues all timestamps
- Simple but single point of failure
- Use Raft/Paxos for HA
```

**Approach 2: Partitioned timestamps**
```
- Each node gets a range: Node A: 1-1000, Node B: 1001-2000
- Timestamps are unique but not globally ordered
- Good for ID generation, not for ordering
```

**Approach 3: HLC (Hybrid Logical Clock)**
```
- Combine physical time with logical counter
- timestamp = (wall_clock, counter)
- If wall clock advances, reset counter
- If wall clock same/behind, increment counter
- Provides monotonicity and approximate wall-clock ordering
```

**Approach 4: Snowflake-style IDs**
```
- 64-bit ID: timestamp (41 bits) + node ID (10 bits) + sequence (12 bits)
- Each node generates unique, roughly time-ordered IDs
- Used by Twitter, Discord
```

For most cases, I'd recommend HLC because it's simple, doesn't require coordination, and provides good-enough ordering.

### L6 (Staff) Questions

**Q5: How does Google Spanner achieve external consistency (linearizability) across globally distributed datacenters?**

**Answer:**
Spanner uses three key innovations:

**1. TrueTime API**
- Returns an interval [earliest, latest] instead of a point in time
- Backed by GPS receivers and atomic clocks in every datacenter
- GPS provides absolute time, atomic clocks maintain stability between GPS updates
- Typical uncertainty: 1-7 milliseconds

**2. Commit-wait protocol**
- When a transaction commits at timestamp T, Spanner waits until TrueTime.now().earliest > T
- This ensures any future transaction will have a timestamp > T
- Guarantees that if transaction A commits before B starts, A's timestamp < B's timestamp

**3. Paxos-based replication**
- Each shard is replicated across datacenters using Paxos
- Leader handles writes, followers handle reads
- Leader lease ensures only one leader at a time

The tradeoff: commit-wait adds latency (1-7ms) to every write transaction. For cross-region transactions, the Paxos round-trips add 100-200ms. This is acceptable for Spanner's use cases (financial systems, AdWords) but too slow for low-latency applications.

**Q6: Design a system that orders events from millions of mobile clients with unreliable connectivity.**

**Answer:**
This is challenging because vector clocks don't scale to millions of clients.

**My approach:**

**1. Client-side: HLC + client ID**
```
Each client generates: (hlc_timestamp, client_id, sequence)
- HLC provides approximate ordering
- client_id ensures uniqueness
- sequence handles offline events
```

**2. Offline handling**
```
- Client queues events locally with HLC timestamps
- When reconnecting, sends batch with original timestamps
- Server uses client's HLC, not arrival time
```

**3. Server-side: Regional ordering**
```
- Events ingested into regional Kafka clusters
- Each region maintains its own ordering
- Cross-region events merged using HLC
```

**4. Conflict resolution**
```
- For concurrent events (same HLC), use deterministic tiebreaker
- client_id comparison, or hash of event content
- Application-specific merge for conflicts
```

**5. Consistency model**
```
- Eventual consistency for most data
- Read-your-writes within same client
- Causal consistency within conversation/thread
```

**Tradeoffs:**
- Approximate ordering (HLC uncertainty + network delays)
- Possible duplicate events (need idempotency)
- Conflicts resolved by application logic

---

## 1️⃣1️⃣ One Clean Mental Summary

Time in distributed systems is fundamentally different from time on a single computer. Physical clocks drift and can jump backwards, making them unreliable for ordering events. Lamport clocks solve ordering by using a simple counter that increments on every event and takes the maximum when receiving messages. Vector clocks extend this to detect concurrent events by maintaining a counter per process. Hybrid Logical Clocks combine physical and logical time, providing bounded-size timestamps that correlate with wall-clock time. Google's TrueTime uses GPS and atomic clocks to provide bounded uncertainty, enabling global transactions. The key insight is that for most distributed systems, we care about order, not actual time. Choose your clock based on your needs: Lamport for simple ordering, vector for conflict detection, HLC for practical production use, and TrueTime only if you have the infrastructure and need global linearizability.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│              DISTRIBUTED CLOCKS CHEAT SHEET                  │
├─────────────────────────────────────────────────────────────┤
│ PHYSICAL CLOCKS                                              │
│   Problem: Drift (~50ms/hour), NTP jumps, no global time    │
│   Use for: Logs, metrics, human-readable timestamps         │
│   Don't use for: Ordering, transactions, distributed locks  │
├─────────────────────────────────────────────────────────────┤
│ LAMPORT CLOCKS                                               │
│   Rules: increment before event, max(local, received) + 1   │
│   Guarantees: A → B implies L(A) < L(B)                     │
│   Limitation: L(A) < L(B) does NOT imply A → B              │
│   Use for: Simple event ordering                            │
├─────────────────────────────────────────────────────────────┤
│ VECTOR CLOCKS                                                │
│   Structure: Array of counters, one per process             │
│   Compare: A < B if all A[i] <= B[i], some A[i] < B[i]     │
│   Concurrent: Neither A < B nor B < A                       │
│   Use for: Conflict detection, small clusters               │
│   Problem: O(N) size, doesn't scale                         │
├─────────────────────────────────────────────────────────────┤
│ HYBRID LOGICAL CLOCKS (HLC)                                  │
│   Structure: (physical_time, logical_counter)               │
│   Size: Fixed 12 bytes                                      │
│   Use for: Production systems, databases                    │
│   Examples: CockroachDB, YugabyteDB                         │
├─────────────────────────────────────────────────────────────┤
│ TRUETIME (Google)                                            │
│   API: now() returns [earliest, latest]                     │
│   Backed by: GPS + atomic clocks                            │
│   Uncertainty: 1-7ms                                        │
│   Use for: Global linearizable transactions                 │
│   Cost: Expensive infrastructure                            │
├─────────────────────────────────────────────────────────────┤
│ KEY INSIGHT                                                  │
│   Distributed systems care about ORDER, not TIME.           │
│   Use logical clocks for ordering.                          │
│   Use physical clocks only for human-readable output.       │
└─────────────────────────────────────────────────────────────┘
```

