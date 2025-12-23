# Distributed Locks

## 0️⃣ Prerequisites

Before diving into distributed locks, you should understand:

- **Distributed Systems** (covered in Phase 1, Topic 01): Systems where multiple computers communicate over unreliable networks
- **Leader Election** (covered in this Phase, Topic 02): How nodes agree on a single coordinator
- **Consensus** (covered in this Phase, Topic 03): How nodes agree on a value
- **Time & Clocks** (covered in this Phase, Topic 01): Why we can't rely on synchronized clocks

**Quick refresher on locks**: In single-machine programming, a lock (mutex) ensures that only one thread can access a critical section at a time. In distributed systems, we need the same guarantee across multiple machines. But distributed locks are much harder because machines can crash, networks can partition, and clocks aren't synchronized.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

Imagine you're running an e-commerce platform with multiple servers. A customer tries to buy the last item in stock:

```
Server A: Check stock for item X → 1 available
Server B: Check stock for item X → 1 available

Server A: Reserve item X → Success
Server B: Reserve item X → Success (oops!)

Both servers sold the same item!
```

This is the **mutual exclusion problem** in distributed systems: how do you ensure only one process can perform a critical operation at a time?

### Why Distributed Locks Are Hard

```
┌─────────────────────────────────────────────────────────────┐
│           WHY DISTRIBUTED LOCKS ARE HARD                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM 1: Process crashes while holding lock              │
│  - Lock is never released                                   │
│  - Other processes wait forever (deadlock)                  │
│                                                              │
│  PROBLEM 2: Network partition                               │
│  - Process thinks it has lock                               │
│  - Lock service thinks lock is released                     │
│  - Another process acquires "same" lock                     │
│                                                              │
│  PROBLEM 3: Clock skew                                      │
│  - Lock has TTL based on time                               │
│  - Process A's clock is slow                                │
│  - Lock expires on server before A thinks it does           │
│  - Process B acquires lock while A still using it           │
│                                                              │
│  PROBLEM 4: GC pauses                                       │
│  - Process holds lock                                       │
│  - Long GC pause (e.g., 30 seconds)                         │
│  - Lock expires during pause                                │
│  - Process resumes, thinks it still has lock                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### What Breaks Without Proper Distributed Locks

**Without distributed locks:**

- **Inventory systems**: Overselling items
- **Payment systems**: Double charges
- **File systems**: Corrupted files from concurrent writes
- **Scheduled jobs**: Same job runs multiple times
- **Leader election**: Multiple leaders
- **Resource allocation**: Conflicts and deadlocks

### Real Examples of the Problem

**The Redlock Controversy**:
In 2016, Martin Kleppmann (author of "Designing Data-Intensive Applications") published a critique of Redis's Redlock algorithm. He showed scenarios where Redlock could fail, leading to multiple processes holding the "same" lock. Antirez (Redis creator) responded, leading to a famous debate about the limits of distributed locking.

**Google Chubby**:
Google built Chubby specifically because they needed reliable distributed locks. Before Chubby, different teams built their own locking mechanisms, often with subtle bugs. Chubby uses Paxos consensus to ensure safety.

**AWS DynamoDB Lock Client**:
Amazon provides a lock client for DynamoDB because distributed locking is such a common need. They explicitly document the limitations and recommend using it only when necessary.

---

## 2️⃣ Intuition and Mental Model

### The Hotel Room Key Analogy

Imagine a hotel with a special room that only one guest can use at a time.

**Simple Lock (Single Server):**
- Front desk has one key
- Guest asks for key, gets it
- Guest returns key when done
- Problem: What if guest loses key? What if front desk burns down?

**Distributed Lock (Multiple Front Desks):**
- Multiple front desks (for reliability)
- Guest must get key from majority of desks
- If one desk is down, others still work
- Problem: What if guest gets key from desk A, but desk B gives key to another guest?

**Fencing Token (Advanced):**
- Each key has a unique, increasing number
- Room only accepts higher-numbered keys
- Even if two guests have keys, room knows which is newer

### The Lock Safety Properties

```
┌─────────────────────────────────────────────────────────────┐
│               DISTRIBUTED LOCK PROPERTIES                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  SAFETY (Mutual Exclusion):                                 │
│  At most one client can hold the lock at any given time.    │
│  This is the most critical property.                        │
│                                                              │
│  LIVENESS (Deadlock Freedom):                               │
│  Eventually, it's always possible to acquire the lock,      │
│  even if the client holding it crashes.                     │
│                                                              │
│  FAULT TOLERANCE:                                           │
│  The lock service continues to work even if some nodes      │
│  fail, as long as a majority are available.                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ How It Works Internally

### Simple Redis Lock (Single Instance)

The simplest distributed lock uses a single Redis instance:

```java
package com.systemdesign.locks;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.params.SetParams;

import java.util.UUID;

/**
 * Simple Redis-based distributed lock.
 * Uses a single Redis instance, not fault-tolerant.
 */
public class SimpleRedisLock {
    
    private final Jedis jedis;
    private final String lockKey;
    private final String lockValue;
    private final long ttlMillis;
    
    public SimpleRedisLock(Jedis jedis, String resourceName, long ttlMillis) {
        this.jedis = jedis;
        this.lockKey = "lock:" + resourceName;
        this.lockValue = UUID.randomUUID().toString();
        this.ttlMillis = ttlMillis;
    }
    
    /**
     * Try to acquire the lock.
     * Returns true if lock was acquired, false otherwise.
     */
    public boolean tryLock() {
        // SET key value NX PX ttl
        // NX = only set if not exists
        // PX = expire in milliseconds
        SetParams params = SetParams.setParams().nx().px(ttlMillis);
        String result = jedis.set(lockKey, lockValue, params);
        return "OK".equals(result);
    }
    
    /**
     * Release the lock.
     * Only releases if we still own it (compare-and-delete).
     */
    public boolean unlock() {
        // Lua script for atomic compare-and-delete
        String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        
        Object result = jedis.eval(script, 
            java.util.List.of(lockKey), 
            java.util.List.of(lockValue));
        
        return Long.valueOf(1).equals(result);
    }
    
    /**
     * Extend the lock TTL (if we still own it).
     */
    public boolean extend(long additionalMillis) {
        String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('pexpire', KEYS[1], ARGV[2])
            else
                return 0
            end
            """;
        
        Object result = jedis.eval(script,
            java.util.List.of(lockKey),
            java.util.List.of(lockValue, String.valueOf(additionalMillis)));
        
        return Long.valueOf(1).equals(result);
    }
}
```

**Why the Lua script for unlock?**

```
Without atomic compare-and-delete:

Client A:
  1. GET lock:resource → "client-a-id" ✓ (I own it)
  2. (GC pause or network delay)
  
Meanwhile, lock expires, Client B acquires it:
  Client B: SET lock:resource "client-b-id" → OK
  
Client A resumes:
  3. DEL lock:resource → Deletes Client B's lock!

With Lua script (atomic):
  Check and delete happen as one operation.
  Client A's delete fails because value changed.
```

### Redlock Algorithm (Multiple Redis Instances)

Redlock uses multiple independent Redis instances for fault tolerance:

```java
package com.systemdesign.locks;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.params.SetParams;

import java.time.Duration;
import java.time.Instant;
import java.util.*;

/**
 * Redlock algorithm implementation.
 * Uses multiple independent Redis instances for fault tolerance.
 * 
 * WARNING: Redlock has known limitations. See Martin Kleppmann's analysis.
 * For critical systems, consider ZooKeeper or database-based locks.
 */
public class Redlock {
    
    private final List<Jedis> redisInstances;
    private final int quorum;
    private final long ttlMillis;
    private final long clockDriftFactor; // milliseconds per second of TTL
    
    private String lockValue;
    private final String lockKey;
    
    public Redlock(List<Jedis> redisInstances, String resourceName, long ttlMillis) {
        this.redisInstances = redisInstances;
        this.quorum = redisInstances.size() / 2 + 1;
        this.lockKey = "lock:" + resourceName;
        this.ttlMillis = ttlMillis;
        this.clockDriftFactor = 2; // 2ms per second of TTL
    }
    
    /**
     * Try to acquire the lock using Redlock algorithm.
     */
    public LockResult tryLock() {
        lockValue = UUID.randomUUID().toString();
        Instant startTime = Instant.now();
        
        int acquiredCount = 0;
        List<Jedis> acquiredInstances = new ArrayList<>();
        
        // Try to acquire lock on all instances
        for (Jedis redis : redisInstances) {
            try {
                if (tryLockInstance(redis)) {
                    acquiredCount++;
                    acquiredInstances.add(redis);
                }
            } catch (Exception e) {
                // Instance might be down, continue
            }
        }
        
        // Calculate elapsed time
        long elapsedMillis = Duration.between(startTime, Instant.now()).toMillis();
        
        // Calculate validity time (remaining TTL minus drift)
        long drift = (ttlMillis * clockDriftFactor) / 1000 + 2;
        long validityTime = ttlMillis - elapsedMillis - drift;
        
        // Check if we got quorum and have enough validity time
        if (acquiredCount >= quorum && validityTime > 0) {
            return new LockResult(true, validityTime);
        }
        
        // Failed to acquire, release any locks we got
        for (Jedis redis : acquiredInstances) {
            try {
                unlockInstance(redis);
            } catch (Exception e) {
                // Best effort release
            }
        }
        
        return new LockResult(false, 0);
    }
    
    /**
     * Release the lock on all instances.
     */
    public void unlock() {
        for (Jedis redis : redisInstances) {
            try {
                unlockInstance(redis);
            } catch (Exception e) {
                // Best effort release
            }
        }
    }
    
    private boolean tryLockInstance(Jedis redis) {
        SetParams params = SetParams.setParams().nx().px(ttlMillis);
        String result = redis.set(lockKey, lockValue, params);
        return "OK".equals(result);
    }
    
    private void unlockInstance(Jedis redis) {
        String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        redis.eval(script, List.of(lockKey), List.of(lockValue));
    }
    
    public record LockResult(boolean acquired, long validityTimeMillis) {}
}
```

**How Redlock Works:**

```
┌─────────────────────────────────────────────────────────────┐
│                   REDLOCK ALGORITHM                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Setup: N independent Redis instances (typically 5)         │
│  Quorum: N/2 + 1 (typically 3)                              │
│                                                              │
│  ACQUIRE:                                                   │
│  1. Get current time T1                                     │
│  2. Try to acquire lock on all N instances sequentially     │
│     - Use same key, same random value, same TTL             │
│     - Use short timeout for each instance                   │
│  3. Get current time T2                                     │
│  4. Calculate elapsed time: T2 - T1                         │
│  5. Lock is acquired if:                                    │
│     - Got lock on >= quorum instances                       │
│     - Elapsed time < TTL (minus clock drift margin)         │
│  6. If acquired, validity time = TTL - elapsed - drift      │
│  7. If not acquired, release all locks                      │
│                                                              │
│  RELEASE:                                                   │
│  - Release lock on all instances (best effort)              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Fencing Tokens

The fundamental problem with distributed locks:

```
Client A acquires lock (token=33)
Client A starts working
Client A has long GC pause
Lock expires (TTL)
Client B acquires lock (token=34)
Client B writes to storage
Client A wakes up, thinks it has lock
Client A writes to storage (overwrites B's work!)
```

**Solution: Fencing Tokens**

```java
package com.systemdesign.locks;

import java.util.concurrent.atomic.AtomicLong;

/**
 * Lock with fencing token.
 * Each lock acquisition gets a monotonically increasing token.
 * Storage systems should reject operations with old tokens.
 */
public class FencedLock {
    
    private final SimpleRedisLock innerLock;
    private final AtomicLong tokenGenerator;
    private Long currentToken;
    
    public FencedLock(SimpleRedisLock innerLock, AtomicLong tokenGenerator) {
        this.innerLock = innerLock;
        this.tokenGenerator = tokenGenerator;
    }
    
    /**
     * Acquire lock and get fencing token.
     */
    public FencingToken tryLock() {
        if (innerLock.tryLock()) {
            currentToken = tokenGenerator.incrementAndGet();
            return new FencingToken(currentToken, true);
        }
        return new FencingToken(0, false);
    }
    
    public void unlock() {
        innerLock.unlock();
        currentToken = null;
    }
    
    public Long getCurrentToken() {
        return currentToken;
    }
    
    public record FencingToken(long token, boolean acquired) {}
}

/**
 * Storage that respects fencing tokens.
 * Rejects writes with tokens older than the last seen token.
 */
class FencedStorage {
    
    private long lastSeenToken = 0;
    private final Object lock = new Object();
    private String data;
    
    /**
     * Write data only if token is valid (not stale).
     */
    public boolean write(long fencingToken, String newData) {
        synchronized (lock) {
            if (fencingToken < lastSeenToken) {
                // Stale token, reject
                System.out.println("Rejected write with stale token " + fencingToken + 
                    " (last seen: " + lastSeenToken + ")");
                return false;
            }
            
            lastSeenToken = fencingToken;
            data = newData;
            return true;
        }
    }
    
    public String read() {
        synchronized (lock) {
            return data;
        }
    }
}
```

**How Fencing Tokens Work:**

```
Client A acquires lock, gets token=33
Client A: write(token=33, "data-A") → Storage accepts, lastSeenToken=33

Client A pauses (GC)
Lock expires
Client B acquires lock, gets token=34
Client B: write(token=34, "data-B") → Storage accepts, lastSeenToken=34

Client A wakes up
Client A: write(token=33, "data-A") → Storage REJECTS (33 < 34)

Data integrity preserved!
```

### ZooKeeper-Based Locks

ZooKeeper provides stronger guarantees through consensus:

```java
package com.systemdesign.locks;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;

import java.util.concurrent.TimeUnit;

/**
 * ZooKeeper-based distributed lock using Apache Curator.
 * Provides stronger guarantees than Redis-based locks.
 */
public class ZooKeeperLock implements AutoCloseable {
    
    private final InterProcessMutex mutex;
    private final String lockPath;
    
    public ZooKeeperLock(CuratorFramework client, String resourceName) {
        this.lockPath = "/locks/" + resourceName;
        this.mutex = new InterProcessMutex(client, lockPath);
    }
    
    /**
     * Acquire the lock, waiting indefinitely.
     */
    public void lock() throws Exception {
        mutex.acquire();
    }
    
    /**
     * Try to acquire the lock with timeout.
     */
    public boolean tryLock(long timeout, TimeUnit unit) throws Exception {
        return mutex.acquire(timeout, unit);
    }
    
    /**
     * Release the lock.
     */
    public void unlock() throws Exception {
        if (mutex.isOwnedByCurrentThread()) {
            mutex.release();
        }
    }
    
    /**
     * Check if current thread holds the lock.
     */
    public boolean isHeldByCurrentThread() {
        return mutex.isOwnedByCurrentThread();
    }
    
    @Override
    public void close() throws Exception {
        unlock();
    }
}
```

**How ZooKeeper Locks Work:**

```
┌─────────────────────────────────────────────────────────────┐
│              ZOOKEEPER LOCK MECHANISM                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Lock path: /locks/my-resource                              │
│                                                              │
│  ACQUIRE:                                                   │
│  1. Create ephemeral sequential node:                       │
│     /locks/my-resource/lock-0000000001                      │
│                                                              │
│  2. Get all children of /locks/my-resource                  │
│     [lock-0000000001, lock-0000000002, ...]                 │
│                                                              │
│  3. If my node has lowest sequence number:                  │
│     → I have the lock!                                      │
│                                                              │
│  4. If not, watch the node just before mine:                │
│     → Wait for notification when it's deleted               │
│                                                              │
│  RELEASE:                                                   │
│  - Delete my ephemeral node                                 │
│  - Or: node auto-deleted if client disconnects              │
│                                                              │
│  WHY IT'S SAFER:                                            │
│  - Ephemeral nodes: auto-cleanup on disconnect              │
│  - Sequential: fair ordering, no thundering herd            │
│  - Watches: efficient waiting                               │
│  - Consensus-backed: survives node failures                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Database-Based Locks

For simpler setups, use database locks:

```java
package com.systemdesign.locks;

import java.sql.*;
import java.time.Instant;
import java.util.UUID;

/**
 * Database-based distributed lock.
 * Uses a database table with unique constraints.
 */
public class DatabaseLock {
    
    private final Connection connection;
    private final String lockName;
    private final String ownerId;
    private final long ttlSeconds;
    
    public DatabaseLock(Connection connection, String lockName, long ttlSeconds) {
        this.connection = connection;
        this.lockName = lockName;
        this.ownerId = UUID.randomUUID().toString();
        this.ttlSeconds = ttlSeconds;
    }
    
    /**
     * Try to acquire the lock.
     */
    public boolean tryLock() throws SQLException {
        // First, clean up expired locks
        cleanupExpiredLocks();
        
        // Try to insert lock record
        String sql = """
            INSERT INTO distributed_locks (lock_name, owner_id, expires_at)
            VALUES (?, ?, ?)
            ON CONFLICT (lock_name) DO NOTHING
            """;
        
        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.setString(1, lockName);
            stmt.setString(2, ownerId);
            stmt.setTimestamp(3, Timestamp.from(
                Instant.now().plusSeconds(ttlSeconds)));
            
            int rows = stmt.executeUpdate();
            return rows > 0;
        }
    }
    
    /**
     * Release the lock.
     */
    public boolean unlock() throws SQLException {
        String sql = "DELETE FROM distributed_locks WHERE lock_name = ? AND owner_id = ?";
        
        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.setString(1, lockName);
            stmt.setString(2, ownerId);
            
            int rows = stmt.executeUpdate();
            return rows > 0;
        }
    }
    
    /**
     * Extend the lock TTL.
     */
    public boolean extend() throws SQLException {
        String sql = """
            UPDATE distributed_locks 
            SET expires_at = ?
            WHERE lock_name = ? AND owner_id = ?
            """;
        
        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.setTimestamp(1, Timestamp.from(
                Instant.now().plusSeconds(ttlSeconds)));
            stmt.setString(2, lockName);
            stmt.setString(3, ownerId);
            
            int rows = stmt.executeUpdate();
            return rows > 0;
        }
    }
    
    /**
     * Clean up expired locks.
     */
    private void cleanupExpiredLocks() throws SQLException {
        String sql = "DELETE FROM distributed_locks WHERE expires_at < NOW()";
        try (Statement stmt = connection.createStatement()) {
            stmt.executeUpdate(sql);
        }
    }
}
```

**SQL Schema:**

```sql
CREATE TABLE distributed_locks (
    lock_name VARCHAR(255) PRIMARY KEY,
    owner_id VARCHAR(255) NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_locks_expires ON distributed_locks(expires_at);
```

---

## 4️⃣ Simulation-First Explanation

Let's trace through distributed lock scenarios.

### Scenario 1: Normal Lock Acquisition and Release

```
Setup: 5 Redis instances, Redlock algorithm

Time 0ms: Client A wants lock "inventory:item-123"
  - Generate value: "client-a-uuid"
  - TTL: 10000ms

Time 1ms: Try Redis 1 → SET lock:inventory:item-123 "client-a-uuid" NX PX 10000
  - Result: OK ✓

Time 2ms: Try Redis 2 → OK ✓
Time 3ms: Try Redis 3 → OK ✓
Time 4ms: Try Redis 4 → OK ✓
Time 5ms: Try Redis 5 → OK ✓

Time 5ms: Calculate validity
  - Acquired: 5 instances (>= quorum of 3) ✓
  - Elapsed: 5ms
  - Clock drift: 10000 * 0.002 + 2 = 22ms
  - Validity: 10000 - 5 - 22 = 9973ms ✓

Client A has lock for ~10 seconds

Time 5000ms: Client A finishes work, releases lock
  - DEL on all 5 instances

Time 5001ms: Client B can now acquire lock
```

### Scenario 2: Lock Holder Crashes

```
Time 0ms: Client A acquires lock (TTL: 10s)
Time 1000ms: Client A crashes!
  - Lock is NOT released
  - Other clients cannot acquire

Time 10001ms: Lock expires on all Redis instances
  - TTL reached
  - Automatic cleanup

Time 10002ms: Client B tries to acquire
  - All instances return OK
  - Client B has lock

No deadlock! TTL ensures liveness.
```

### Scenario 3: Network Partition During Lock Hold

```
Time 0ms: Client A acquires lock on Redis 1,2,3,4,5
  - Validity: 10s

Time 1000ms: Network partition!
  - Client A can only reach Redis 1,2
  - Redis 3,4,5 are unreachable

Time 5000ms: Client A's lock expires on Redis 3,4,5
  - Client A doesn't know this!

Time 5001ms: Client B tries to acquire
  - Redis 1,2: locked by A (can't acquire)
  - Redis 3,4,5: expired (can acquire)
  - Client B gets 3 locks (quorum!)
  - Client B thinks it has the lock

Time 5002ms: BOTH A and B think they have the lock!
  - This is the Redlock safety violation
  - Fencing tokens would prevent damage

LESSON: Redlock is not safe under network partitions.
Use fencing tokens or ZooKeeper for critical sections.
```

### Scenario 4: Fencing Token Saves the Day

```
Time 0ms: Client A acquires lock, token=100
Time 1ms: Client A: write(token=100, "A") → Accepted

Time 1000ms: Client A has long GC pause

Time 5000ms: Lock expires
Time 5001ms: Client B acquires lock, token=101
Time 5002ms: Client B: write(token=101, "B") → Accepted

Time 10000ms: Client A wakes up from GC
Time 10001ms: Client A: write(token=100, "A") → REJECTED!
  - Storage sees token=100 < lastSeen=101
  - Stale write prevented

Data integrity maintained!
```

---

## 5️⃣ How Engineers Actually Use This in Production

### At Major Companies

**Google (Chubby)**:
- Built specifically for distributed locking
- Uses Paxos for consensus
- Provides coarse-grained locks (held for minutes/hours)
- Used by BigTable, GFS for coordination

**Amazon (DynamoDB Lock Client)**:
- Provides lock client library
- Uses DynamoDB table for lock storage
- Heartbeat-based lease renewal
- Explicitly documents limitations

**Netflix**:
- Uses ZooKeeper for coordination
- Curator library for lock recipes
- Fallback to database locks for simpler cases

**Uber**:
- Uses Redis for high-performance locks
- ZooKeeper for critical coordination
- Custom fencing for payment systems

### Production Checklist

```
┌─────────────────────────────────────────────────────────────┐
│           DISTRIBUTED LOCK PRODUCTION CHECKLIST              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  DESIGN                                                     │
│  □ Do you actually need a distributed lock?                 │
│  □ Can you use optimistic locking instead?                  │
│  □ Can you partition the problem to avoid locks?            │
│                                                              │
│  IMPLEMENTATION                                             │
│  □ Set appropriate TTL (not too short, not too long)        │
│  □ Use fencing tokens for storage operations                │
│  □ Handle lock acquisition failure gracefully               │
│  □ Implement lock extension for long operations             │
│                                                              │
│  SAFETY                                                     │
│  □ Test with network partitions                             │
│  □ Test with process crashes                                │
│  □ Test with clock skew                                     │
│  □ Test with GC pauses                                      │
│                                                              │
│  MONITORING                                                 │
│  □ Track lock acquisition time                              │
│  □ Track lock hold time                                     │
│  □ Alert on lock contention                                 │
│  □ Alert on lock timeouts                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### When to Use Which Lock

```
┌─────────────────────────────────────────────────────────────┐
│              DISTRIBUTED LOCK DECISION TREE                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Is mutual exclusion critical (data corruption if broken)?  │
│  ├── YES → Use ZooKeeper + fencing tokens                   │
│  │         Or: Database with transactions                   │
│  │                                                          │
│  └── NO → Is performance critical?                          │
│           ├── YES → Use Redis (single instance or Redlock)  │
│           │         Accept occasional duplicates            │
│           │                                                 │
│           └── NO → Use database lock                        │
│                    Simple, debuggable, durable              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6️⃣ How to Implement or Apply It

### Complete Spring Boot Implementation

#### Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-redis</artifactId>
    </dependency>
    <!-- For ZooKeeper locks -->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-recipes</artifactId>
        <version>5.5.0</version>
    </dependency>
</dependencies>
```

#### Distributed Lock Service

```java
package com.systemdesign.locks;

import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

/**
 * Production-ready distributed lock service.
 */
@Service
public class DistributedLockService {
    
    private final StringRedisTemplate redis;
    
    private static final String UNLOCK_SCRIPT = """
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('del', KEYS[1])
        else
            return 0
        end
        """;
    
    private static final String EXTEND_SCRIPT = """
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('pexpire', KEYS[1], ARGV[2])
        else
            return 0
        end
        """;
    
    public DistributedLockService(StringRedisTemplate redis) {
        this.redis = redis;
    }
    
    /**
     * Execute a task with a distributed lock.
     * Automatically acquires, executes, and releases.
     */
    public <T> T executeWithLock(String resourceName, Duration timeout, 
                                  Duration lockTtl, Supplier<T> task) {
        String lockKey = "lock:" + resourceName;
        String lockValue = UUID.randomUUID().toString();
        
        // Try to acquire lock with timeout
        long deadline = System.currentTimeMillis() + timeout.toMillis();
        boolean acquired = false;
        
        while (System.currentTimeMillis() < deadline) {
            acquired = tryAcquire(lockKey, lockValue, lockTtl);
            if (acquired) break;
            
            try {
                Thread.sleep(50); // Wait before retry
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new LockException("Interrupted while waiting for lock");
            }
        }
        
        if (!acquired) {
            throw new LockException("Failed to acquire lock within timeout");
        }
        
        try {
            return task.get();
        } finally {
            release(lockKey, lockValue);
        }
    }
    
    /**
     * Execute with lock and automatic extension.
     * For long-running tasks that might exceed TTL.
     */
    public <T> T executeWithAutoExtend(String resourceName, Duration lockTtl,
                                        Supplier<T> task) {
        String lockKey = "lock:" + resourceName;
        String lockValue = UUID.randomUUID().toString();
        
        if (!tryAcquire(lockKey, lockValue, lockTtl)) {
            throw new LockException("Failed to acquire lock");
        }
        
        // Start background thread to extend lock
        Thread extender = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    Thread.sleep(lockTtl.toMillis() / 3);
                    if (!extend(lockKey, lockValue, lockTtl)) {
                        break; // Lost the lock
                    }
                } catch (InterruptedException e) {
                    break;
                }
            }
        });
        extender.setDaemon(true);
        extender.start();
        
        try {
            return task.get();
        } finally {
            extender.interrupt();
            release(lockKey, lockValue);
        }
    }
    
    private boolean tryAcquire(String key, String value, Duration ttl) {
        Boolean result = redis.opsForValue()
            .setIfAbsent(key, value, ttl);
        return Boolean.TRUE.equals(result);
    }
    
    private boolean release(String key, String value) {
        DefaultRedisScript<Long> script = new DefaultRedisScript<>(UNLOCK_SCRIPT, Long.class);
        Long result = redis.execute(script, List.of(key), value);
        return Long.valueOf(1).equals(result);
    }
    
    private boolean extend(String key, String value, Duration ttl) {
        DefaultRedisScript<Long> script = new DefaultRedisScript<>(EXTEND_SCRIPT, Long.class);
        Long result = redis.execute(script, List.of(key), value, 
            String.valueOf(ttl.toMillis()));
        return Long.valueOf(1).equals(result);
    }
    
    public static class LockException extends RuntimeException {
        public LockException(String message) {
            super(message);
        }
    }
}
```

#### Usage Example

```java
package com.systemdesign.locks;

import org.springframework.stereotype.Service;

import java.time.Duration;

@Service
public class InventoryService {
    
    private final DistributedLockService lockService;
    private final InventoryRepository inventoryRepository;
    
    public InventoryService(DistributedLockService lockService,
                            InventoryRepository inventoryRepository) {
        this.lockService = lockService;
        this.inventoryRepository = inventoryRepository;
    }
    
    /**
     * Reserve inventory with distributed lock.
     */
    public boolean reserveInventory(String productId, int quantity) {
        return lockService.executeWithLock(
            "inventory:" + productId,
            Duration.ofSeconds(5),   // Wait up to 5s for lock
            Duration.ofSeconds(10),  // Lock TTL 10s
            () -> {
                int available = inventoryRepository.getAvailable(productId);
                if (available >= quantity) {
                    inventoryRepository.reserve(productId, quantity);
                    return true;
                }
                return false;
            }
        );
    }
    
    /**
     * Long-running inventory reconciliation with auto-extend.
     */
    public void reconcileInventory(String warehouseId) {
        lockService.executeWithAutoExtend(
            "reconciliation:" + warehouseId,
            Duration.ofMinutes(5),
            () -> {
                // Long-running reconciliation
                // Lock is automatically extended
                performReconciliation(warehouseId);
                return null;
            }
        );
    }
    
    private void performReconciliation(String warehouseId) {
        // Implementation
    }
    
    interface InventoryRepository {
        int getAvailable(String productId);
        void reserve(String productId, int quantity);
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

#### 1. Not Using Unique Lock Values

**Wrong:**
```java
// BAD: Same value for all locks
redis.set("lock:resource", "locked", SetParams.setParams().nx().px(10000));

// Later, any client can delete it!
redis.del("lock:resource");
```

**Right:**
```java
// GOOD: Unique value per lock acquisition
String lockValue = UUID.randomUUID().toString();
redis.set("lock:resource", lockValue, SetParams.setParams().nx().px(10000));

// Only delete if we own it
String script = "if redis.call('get',KEYS[1])==ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end";
redis.eval(script, List.of("lock:resource"), List.of(lockValue));
```

#### 2. TTL Too Short or Too Long

**Wrong:**
```java
// BAD: TTL too short
Duration lockTtl = Duration.ofMillis(100);
// Lock expires before work is done!

// BAD: TTL too long
Duration lockTtl = Duration.ofHours(1);
// If process crashes, resource is locked for an hour!
```

**Right:**
```java
// GOOD: Reasonable TTL with extension
Duration lockTtl = Duration.ofSeconds(30);
// Plus: background thread extends if needed
```

#### 3. Not Handling Lock Failure

**Wrong:**
```java
// BAD: Assumes lock always succeeds
public void process(String resourceId) {
    lockService.lock(resourceId);
    doWork();
    lockService.unlock(resourceId);
}
```

**Right:**
```java
// GOOD: Handle lock failure
public void process(String resourceId) {
    if (!lockService.tryLock(resourceId, Duration.ofSeconds(5))) {
        throw new ResourceBusyException("Could not acquire lock");
    }
    try {
        doWork();
    } finally {
        lockService.unlock(resourceId);
    }
}
```

#### 4. Trusting the Lock Without Fencing

**Wrong:**
```java
// BAD: No fencing token
lock.acquire();
storage.write(data); // Might execute after lock expired!
lock.release();
```

**Right:**
```java
// GOOD: Use fencing token
FencingToken token = lock.acquire();
storage.writeWithFencing(token, data);
lock.release();
```

### Performance Considerations

```
┌─────────────────────────────────────────────────────────────┐
│           DISTRIBUTED LOCK PERFORMANCE                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Redis (single instance):                                   │
│  - Acquire: ~1ms                                            │
│  - Release: ~1ms                                            │
│  - Throughput: ~100K locks/sec                              │
│                                                              │
│  Redlock (5 instances):                                     │
│  - Acquire: ~5-10ms                                         │
│  - Release: ~5ms                                            │
│  - Throughput: ~10K locks/sec                               │
│                                                              │
│  ZooKeeper:                                                 │
│  - Acquire: ~10-50ms                                        │
│  - Release: ~10ms                                           │
│  - Throughput: ~1K locks/sec                                │
│                                                              │
│  Database:                                                  │
│  - Acquire: ~5-20ms                                         │
│  - Release: ~5ms                                            │
│  - Throughput: ~1K locks/sec                                │
│                                                              │
│  OPTIMIZATION:                                              │
│  - Batch operations when possible                           │
│  - Use local caching for read-heavy locks                   │
│  - Partition locks by resource type                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8️⃣ When NOT to Use This

### When Distributed Locks Are Unnecessary

1. **Single-threaded processing**: No concurrency, no lock needed
2. **Idempotent operations**: Duplicates are harmless
3. **Optimistic locking works**: Version numbers prevent conflicts
4. **Partitioned data**: Each partition has its own owner

### When Simpler Alternatives Work

1. **Database transactions**: ACID guarantees mutual exclusion
2. **Optimistic locking**: Compare-and-swap at storage level
3. **Queue-based processing**: Single consumer per resource
4. **Actor model**: Each actor owns its state

### Anti-Patterns

1. **Lock for everything**
   - Locks are expensive and complex
   - Design to avoid locks when possible

2. **Long-held locks**
   - Increases contention
   - Higher risk of expiration issues

3. **Nested locks**
   - Risk of deadlock
   - Hard to reason about

---

## 9️⃣ Comparison with Alternatives

### Lock Implementation Comparison

| Aspect | Redis Single | Redlock | ZooKeeper | Database |
|--------|--------------|---------|-----------|----------|
| Safety | Low | Medium | High | High |
| Performance | High | Medium | Low | Low |
| Complexity | Low | Medium | High | Low |
| Fault tolerance | None | Partial | High | Depends |
| Fencing support | Manual | Manual | Built-in | Manual |

### When to Use Each

| Use Case | Recommended |
|----------|-------------|
| High-performance, best-effort | Redis single |
| Balanced safety/performance | Redlock + fencing |
| Critical sections | ZooKeeper |
| Simple setups | Database |
| Already have ZK | ZooKeeper |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q1: What is a distributed lock and why do we need it?**

**Answer:**
A distributed lock ensures that only one process across multiple machines can access a shared resource at a time. We need it because:

1. **Mutual exclusion**: Prevent race conditions when multiple servers access shared resources (like inventory or account balances).

2. **Coordination**: Ensure only one instance runs a scheduled job or becomes the leader.

Unlike single-machine locks, distributed locks are harder because:
- Processes can crash while holding locks
- Networks can partition
- Clocks aren't synchronized

Example: Two servers checking inventory might both see "1 item available" and both try to sell it, resulting in overselling.

**Q2: What is a fencing token and why is it important?**

**Answer:**
A fencing token is a monotonically increasing number assigned with each lock acquisition. Storage systems should reject operations with tokens lower than the last seen token.

It's important because:

1. **GC pauses**: A process might pause for seconds (garbage collection), during which its lock expires and another process acquires it. When the first process resumes, it still thinks it has the lock.

2. **Network delays**: A process might send a write request, then lose the lock, but the request is still in flight.

Without fencing tokens, stale lock holders can corrupt data. With fencing tokens, storage rejects stale writes.

### L5 (Senior) Questions

**Q3: Explain the Redlock algorithm and its limitations.**

**Answer:**
**Redlock algorithm:**
1. Get current time
2. Try to acquire lock on N independent Redis instances (typically 5)
3. Calculate elapsed time
4. Lock is acquired if: majority acquired AND elapsed < TTL
5. Validity time = TTL - elapsed - clock drift margin
6. If not acquired, release all locks

**Limitations (from Martin Kleppmann's analysis):**

1. **Clock assumptions**: Redlock assumes bounded clock drift. If a Redis node's clock jumps forward, the lock might expire early.

2. **Process pauses**: A client might acquire the lock, then pause (GC). When it resumes, the lock has expired but it doesn't know.

3. **Network delays**: Similar issue with delayed messages.

4. **Not consensus-based**: Unlike ZooKeeper, Redlock doesn't use consensus. It's possible for two clients to both believe they have the lock.

**Recommendation**: For critical sections, use ZooKeeper with fencing tokens. Use Redlock only for efficiency (preventing duplicate work) where occasional duplicates are acceptable.

**Q4: How would you implement a distributed lock that survives process crashes?**

**Answer:**
Key mechanisms:

1. **TTL (Time-To-Live)**: Lock automatically expires if not renewed. Prevents permanent deadlock if holder crashes.

2. **Lease renewal**: Long-running operations extend the lock before TTL expires. Background thread renews every TTL/3.

3. **Unique owner ID**: Each lock acquisition has a unique ID. Only the owner can release or extend.

4. **Atomic operations**: Use Lua scripts in Redis or transactions in databases for check-and-modify operations.

5. **Fencing tokens**: Even if a crashed process's lock expires and another acquires it, fencing tokens prevent the crashed process from causing damage when it recovers.

Implementation approach:
```
acquire():
  1. Generate unique owner ID
  2. SET key owner_id NX PX ttl
  3. Start background renewal thread
  4. Return fencing token

release():
  1. Stop renewal thread
  2. Atomic: IF key == owner_id THEN DELETE
```

### L6 (Staff) Questions

**Q5: Design a distributed lock service for a global e-commerce platform.**

**Answer:**
Requirements:
- High availability across regions
- Low latency for lock acquisition
- Strong safety guarantees for payments
- Reasonable performance for inventory

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│            GLOBAL DISTRIBUTED LOCK SERVICE                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  TIER 1: Regional Redis (High Performance)                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ US-East     │  │ US-West     │  │ Europe      │         │
│  │ Redis       │  │ Redis       │  │ Redis       │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│       │                │                │                   │
│       └────────────────┼────────────────┘                   │
│                        │                                    │
│  TIER 2: Global ZooKeeper (High Safety)                     │
│  ┌─────────────────────────────────────────────────┐       │
│  │ ZooKeeper Ensemble (5 nodes across 3 regions)   │       │
│  └─────────────────────────────────────────────────┘       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Design decisions:**

1. **Two-tier approach**:
   - Tier 1 (Redis): Fast locks for inventory, cart operations
   - Tier 2 (ZooKeeper): Strong locks for payments, critical operations

2. **Regional affinity**: Route lock requests to nearest region for latency

3. **Fencing tokens**: All storage operations require fencing tokens

4. **Lock hierarchy**: 
   - Global locks for cross-region resources
   - Regional locks for region-specific resources

5. **Monitoring**: Track lock contention, acquisition time, expiration rate

**Trade-offs:**
- Tier 1: Fast but might have rare duplicates
- Tier 2: Slow (100-200ms cross-region) but safe

**Q6: What's the difference between distributed locks and optimistic locking? When would you use each?**

**Answer:**

**Distributed Locks (Pessimistic):**
- Acquire lock BEFORE reading/modifying
- Block other processes
- Guarantee exclusive access
- Higher contention, lower throughput

**Optimistic Locking:**
- Read data with version number
- Modify locally
- Write back with version check
- If version changed, retry
- No blocking, higher throughput

**When to use distributed locks:**
- Long-running operations
- External side effects (can't retry easily)
- High contention (many retries with optimistic)
- Need to hold multiple resources atomically

**When to use optimistic locking:**
- Short operations
- Low contention
- Easy to retry
- Single resource updates

**Hybrid approach:**
```java
// Try optimistic first
for (int i = 0; i < 3; i++) {
    if (optimisticUpdate()) return;
}
// Fall back to distributed lock
distributedLock.executeWithLock(() -> {
    forceUpdate();
});
```

---

## 1️⃣1️⃣ One Clean Mental Summary

Distributed locks provide mutual exclusion across multiple machines, ensuring only one process can access a critical resource at a time. The key challenges are process crashes (solved with TTL), network partitions (solved with quorum), and stale lock holders (solved with fencing tokens). Redis provides fast but less safe locks; ZooKeeper provides safe but slower locks. The Redlock algorithm attempts to balance safety and performance but has known limitations under network partitions. Fencing tokens are essential: they allow storage systems to reject stale operations even if the lock holder doesn't know its lock has expired. Before using distributed locks, consider if you can avoid them through optimistic locking, partitioning, or idempotent operations. When you do need them, choose the right tool based on your safety requirements: Redis for best-effort, ZooKeeper for critical sections.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│            DISTRIBUTED LOCKS CHEAT SHEET                     │
├─────────────────────────────────────────────────────────────┤
│ PROPERTIES                                                   │
│   Safety: At most one holder at a time                      │
│   Liveness: Lock eventually becomes available               │
│   Fault tolerance: Survives node failures                   │
├─────────────────────────────────────────────────────────────┤
│ REDIS SINGLE INSTANCE                                        │
│   Acquire: SET key value NX PX ttl                          │
│   Release: Lua script (check value, then delete)            │
│   Safety: Low (single point of failure)                     │
│   Performance: High (~100K/sec)                             │
├─────────────────────────────────────────────────────────────┤
│ REDLOCK (N instances)                                        │
│   Acquire: Lock on N instances, need majority               │
│   Validity: TTL - elapsed - clock_drift                     │
│   Safety: Medium (has known limitations)                    │
│   Performance: Medium (~10K/sec)                            │
├─────────────────────────────────────────────────────────────┤
│ ZOOKEEPER                                                    │
│   Acquire: Create ephemeral sequential node                 │
│   Release: Delete node (or auto on disconnect)              │
│   Safety: High (consensus-based)                            │
│   Performance: Low (~1K/sec)                                │
├─────────────────────────────────────────────────────────────┤
│ FENCING TOKENS                                               │
│   Each acquisition gets increasing token                    │
│   Storage rejects writes with old tokens                    │
│   Prevents stale lock holders from causing damage           │
├─────────────────────────────────────────────────────────────┤
│ COMMON MISTAKES                                              │
│   ✗ No unique lock value (can't verify ownership)           │
│   ✗ TTL too short (expires during work)                     │
│   ✗ TTL too long (long deadlock on crash)                   │
│   ✗ No fencing tokens (stale holders corrupt data)          │
│   ✗ Trusting lock without verification                      │
└─────────────────────────────────────────────────────────────┘
```

