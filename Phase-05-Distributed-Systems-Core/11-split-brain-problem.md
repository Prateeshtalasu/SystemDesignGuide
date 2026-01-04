# 🧠 Split Brain Problem

## 0️⃣ Prerequisites

Before diving into the split brain problem, you should understand:

- **Distributed Systems Basics**: Multiple nodes working together over a network
- **Network Partitions**: When network failures prevent some nodes from communicating with others
- **Leader Election**: The process of selecting one node to coordinate operations (covered in Topic 2)
- **Consensus**: Agreement among distributed nodes on a single value (covered in Topic 3)
- **Replication**: Copying data across multiple nodes for availability and durability

**Quick refresher**: In a distributed system, nodes communicate over a network. Networks can fail, creating "partitions" where some nodes can talk to each other but not to other groups. When this happens, each group might think it's the only one still running, leading to the "split brain" problem.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

Imagine a database cluster with 3 nodes. Node A is the leader (handles writes). Nodes B and C are followers (replicate from A).

**Normal operation**:
```
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Node A  │────►│ Node B  │     │ Node C  │
│(Leader) │     │(Follower)│◄───│(Follower)│
└─────────┘     └─────────┘     └─────────┘
     │               ▲               ▲
     └───────────────┴───────────────┘
           Replication
```

**Network partition occurs**:
```
┌─────────┐          ║          ┌─────────┐
│ Node A  │          ║          │ Node B  │
│(Leader) │    PARTITION        │(Follower)│
└─────────┘          ║          └─────────┘
                     ║               │
                     ║          ┌─────────┐
                     ║          │ Node C  │
                     ║          │(Follower)│
                     ║          └─────────┘
```

**The split brain problem**:
- Node A is isolated but thinks it's still the leader
- Nodes B and C can't reach A, so they elect B as new leader
- Now TWO nodes think they're the leader
- Both accept writes independently
- Data diverges and becomes inconsistent

```
Node A (thinks it's leader):     Node B (elected as new leader):
  Write: x = 100                   Write: x = 200
  Write: y = "foo"                 Write: y = "bar"

When partition heals:
  x = ??? (100 or 200?)
  y = ??? ("foo" or "bar"?)
```

### What Systems Looked Like Before

**Era 1: Single Node**
- No split brain (only one node)
- But no fault tolerance either

**Era 2: Primary-Backup (No Automatic Failover)**
- Manual intervention required
- Operator decides which node is primary
- Safe but slow recovery

**Era 3: Automatic Failover (Naive)**
- Automatic leader election on failure
- But vulnerable to split brain
- Multiple leaders possible

**Era 4: Quorum-Based Systems**
- Majority required for decisions
- Prevents split brain
- But requires careful design

### What Breaks Without Split Brain Prevention

| Scenario | Impact |
|----------|--------|
| Dual leaders | Conflicting writes, data corruption |
| Data divergence | Inconsistent reads, application errors |
| Transaction conflicts | Lost updates, duplicate transactions |
| Financial systems | Double charges, incorrect balances |
| Inventory systems | Overselling, negative stock |

### Real Examples of the Problem

**GitHub's 2012 Outage**: A network partition caused their MySQL cluster to have two masters. Both accepted writes, leading to data inconsistency. Recovery took hours and required manual reconciliation.

**MongoDB Split Brain**: Early MongoDB versions were vulnerable to split brain during network partitions. This led to data loss in production systems, prompting major architectural changes.

**RabbitMQ Cluster Split**: RabbitMQ clusters could split during network issues, with each partition continuing to serve messages. Messages could be lost or duplicated when the partition healed.

---

## 2️⃣ Intuition and Mental Model

### The Two Generals Analogy

Imagine two generals with armies on opposite sides of an enemy city. They need to attack simultaneously or they'll lose. They can only communicate by sending messengers through enemy territory.

```
General A ────────── Enemy City ────────── General B
              (messengers may be captured)
```

**The problem**:
- General A sends: "Attack at dawn"
- Did General B receive it?
- General B sends: "Confirmed"
- Did General A receive the confirmation?
- Neither can be 100% sure the other got the message

**This is the fundamental challenge**: In a distributed system with unreliable communication, you can never be certain about the state of other nodes.

### The Brain Surgery Analogy

Think of a distributed system as a brain with two hemispheres:

**Normal brain**:
- Left and right hemispheres communicate
- Coordinated decisions
- One coherent consciousness

**Split brain (medical condition)**:
- Corpus callosum severed
- Hemispheres can't communicate
- Each hemisphere acts independently
- Conflicting decisions

**The distributed system parallel**:
- Network partition = severed connection
- Each partition thinks it's "the brain"
- Both make independent decisions
- Conflicts when reconnected

### The Key Insight

Split brain is fundamentally about **conflicting authority**. When nodes can't communicate, they can't coordinate. Without coordination, multiple nodes may assume authority, leading to conflicts.

**Prevention strategies** all revolve around one principle: **Only allow authority when you can prove you have it**.

---

## 3️⃣ How It Works Internally

### Causes of Split Brain

**1. Network Partition**
```
┌─────────────┐     ┌─────────────┐
│ Datacenter A│  X  │ Datacenter B│
│   Node 1    │     │   Node 3    │
│   Node 2    │     │   Node 4    │
└─────────────┘     └─────────────┘
      Network link fails
```

**2. Node Isolation**
```
┌─────────────────────────────────┐
│         Network                  │
│    ┌───┐   ┌───┐   ┌───┐       │
│    │ A │───│ B │───│ C │       │
│    └───┘   └───┘   └───┘       │
│      │                          │
│      X (A's network card fails) │
│                                 │
└─────────────────────────────────┘
Node A is isolated but still running
```

**3. Asymmetric Partition**
```
     A ──────► B    (A can send to B)
     A ◄────X─ B    (B cannot send to A)
     
A thinks B is dead (no responses)
B thinks A is alive (receiving messages)
```

**4. GC Pause / Process Hang**
```
Node A: Long garbage collection pause (30 seconds)
Other nodes: A appears dead, elect new leader
Node A: GC completes, thinks it's still leader
Result: Two leaders
```

### Detection Mechanisms

**Heartbeat Timeout**
```java
class HeartbeatMonitor {
    private long lastHeartbeat;
    private final long timeoutMs;
    
    boolean isAlive() {
        return System.currentTimeMillis() - lastHeartbeat < timeoutMs;
    }
    
    void onHeartbeatReceived() {
        lastHeartbeat = System.currentTimeMillis();
    }
}
```

**Problem**: Can't distinguish between:
- Node actually dead
- Network partition
- Node temporarily slow (GC, high load)

**Lease-Based Detection**
```java
class LeaderLease {
    private final long leaseStartTime;
    private final long leaseDurationMs;
    
    boolean isLeaseValid() {
        return System.currentTimeMillis() < leaseStartTime + leaseDurationMs;
    }
    
    // Leader must renew lease before expiry
    // If can't renew (partition), lease expires
    // Leader MUST stop acting as leader when lease expires
}
```

### Prevention Strategies

#### Strategy 1: Quorum-Based Decisions

**Principle**: Require majority agreement for any decision.

```
5-node cluster:
- Quorum = 3 (majority)
- Any partition with < 3 nodes cannot make decisions
- At most ONE partition can have quorum

Partition scenario:
[A, B] | [C, D, E]
  2    |    3
       
[A, B]: Cannot form quorum, stops accepting writes
[C, D, E]: Has quorum, elects new leader, continues
```

**Why it works**:
- Two disjoint sets cannot both have majority
- Only one partition can have quorum
- Only one partition can make decisions

#### Strategy 2: Fencing Tokens

**Principle**: Use monotonically increasing tokens to order operations.

```
Epoch 1: Leader A gets token 1
         A's writes tagged with token 1
         
Partition occurs...

Epoch 2: Leader B elected, gets token 2
         B's writes tagged with token 2
         
A tries to write with token 1
Storage rejects: token 1 < current token 2
```

```java
class FencedStorage {
    private long currentToken = 0;
    
    void write(long token, String key, String value) {
        if (token < currentToken) {
            throw new FencedOffException("Stale token: " + token);
        }
        currentToken = token;
        storage.put(key, value);
    }
}
```

#### Strategy 3: STONITH (Shoot The Other Node In The Head)

**Principle**: When in doubt, forcibly terminate the other node.

```
Node B detects Node A might be dead:
1. B cannot reach A via network
2. B sends STONITH command via out-of-band channel:
   - Power management (IPMI, iLO, DRAC)
   - Shared storage reservation
   - Network switch port disable
3. B confirms A is definitely dead
4. B safely becomes leader
```

**Why it works**:
- Eliminates uncertainty about A's state
- A is guaranteed not to be running
- No possibility of dual leaders

**Risks**:
- Aggressive (kills healthy nodes on network issues)
- Requires out-of-band management network
- Can cause unnecessary downtime

#### Strategy 4: Leader Lease

**Principle**: Leader authority expires if not renewed.

```
Leader A acquires lease for 10 seconds
A must renew lease every 5 seconds

If A can't reach quorum to renew:
- Lease expires after 10 seconds
- A MUST stop acting as leader
- Other nodes can elect new leader after 10 seconds
```

```java
class LeaderLease {
    private volatile long leaseExpiry;
    private final long leaseDurationMs = 10000;
    private final long renewIntervalMs = 5000;
    
    boolean tryRenewLease() {
        if (quorum.renewLease()) {
            leaseExpiry = System.currentTimeMillis() + leaseDurationMs;
            return true;
        }
        return false;
    }
    
    boolean canActAsLeader() {
        // Must have valid lease AND buffer time
        return System.currentTimeMillis() < leaseExpiry - clockDriftBuffer;
    }
}
```

**Critical**: Leader must account for clock drift. If leader's clock is fast, it might think lease is valid when it's actually expired on other nodes.

---

## 4️⃣ Simulation-First Explanation

Let's simulate a split brain scenario and prevention.

### Setup

```
3-node cluster: A, B, C
Node A is current leader
Replication: A → B, A → C
```

### Scenario 1: Split Brain Without Prevention

**T=0: Normal operation**
```
Client writes: x = 100
A (leader): Accepts write, replicates to B, C
State: A.x=100, B.x=100, C.x=100
```

**T=10: Network partition**
```
[A] | [B, C]
A isolated from B and C
B and C can communicate
```

**T=15: Both partitions active**
```
Partition 1 [A]:
- A doesn't know about partition
- A still thinks it's leader
- Client 1 writes: x = 200
- A accepts: A.x = 200

Partition 2 [B, C]:
- B and C can't reach A
- After timeout, B becomes leader
- Client 2 writes: x = 300
- B accepts: B.x = 300, C.x = 300
```

**T=30: Partition heals**
```
Network restored
A.x = 200
B.x = 300
C.x = 300

CONFLICT! Which value is correct?
Data corruption has occurred.
```

### Scenario 2: Split Brain Prevention with Quorum

**T=0: Normal operation**
```
3-node cluster, quorum = 2
A is leader
```

**T=10: Network partition**
```
[A] | [B, C]
```

**T=15: Quorum check**
```
Partition 1 [A]:
- A tries to write x = 200
- A checks quorum: can only reach self (1 node)
- 1 < quorum (2)
- A REJECTS write
- A steps down as leader

Partition 2 [B, C]:
- B and C can't reach A
- They have 2 nodes (= quorum)
- B becomes leader
- Client writes x = 300
- B replicates to C
- Write succeeds
```

**T=30: Partition heals**
```
A rejoins cluster
A sees B is leader with higher term
A becomes follower
A syncs: A.x = 300

All nodes consistent: x = 300
No split brain!
```

### Scenario 3: Fencing Token Prevention

**T=0: Normal operation**
```
A is leader with fencing token = 1
Shared storage knows current token = 1
```

**T=10: Network partition**
```
[A] | [B, C]
```

**T=15: New leader election**
```
Partition 2 [B, C]:
- B becomes leader
- B gets fencing token = 2
- Storage updated: current token = 2

Partition 1 [A]:
- A still has token = 1
- A tries to write to storage
- Storage rejects: token 1 < current token 2
- A's writes fail
```

**T=20: A realizes it's fenced**
```
A's writes keep failing
A checks its token (1) vs storage token (2)
A realizes it's been fenced off
A steps down
```

**T=30: Partition heals**
```
A rejoins as follower
Single leader (B) with token 2
No split brain!
```

### Scenario 4: STONITH Prevention

**T=0: Normal operation**
```
A is leader
A, B, C all have IPMI access to each other
```

**T=10: Network partition**
```
[A] | [B, C]
```

**T=15: STONITH decision**
```
Partition 2 [B, C]:
- B and C can't reach A
- Before electing new leader, STONITH A
- B sends IPMI power-off command to A
- A is forcibly shut down
- B confirms A is off
- B becomes leader safely
```

**T=20: A is dead**
```
A is powered off
B is only leader
No possibility of split brain
```

**T=30: Operator intervention**
```
Operator investigates why A was STONITHed
Operator fixes network issue
Operator powers A back on
A rejoins as follower
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Apache ZooKeeper

**How ZooKeeper prevents split brain**:

1. **Quorum-based**: Requires majority for all decisions
2. **Leader election**: Uses Zab protocol (similar to Paxos)
3. **Ephemeral nodes**: Automatically deleted when session expires

```java
// ZooKeeper leader election
public class LeaderElection {
    private ZooKeeper zk;
    
    public void electLeader() {
        // Create ephemeral sequential node
        String path = zk.create("/election/node_", 
            data, EPHEMERAL_SEQUENTIAL);
        
        // Get all candidates
        List<String> children = zk.getChildren("/election", false);
        Collections.sort(children);
        
        // Lowest sequence number is leader
        if (path.endsWith(children.get(0))) {
            becomeLeader();
        } else {
            watchPredecessor(children);
        }
    }
}
```

**If partition occurs**:
- Minority partition loses ZooKeeper session
- Ephemeral nodes deleted
- Minority cannot act as leader

### etcd and Raft

**How etcd prevents split brain**:

1. **Raft consensus**: Leader needs majority for commits
2. **Term numbers**: Fencing via monotonic terms
3. **Leader lease**: Leader steps down if can't reach majority

```
Raft term mechanism:
- Each election increments term
- Higher term always wins
- Old leader with lower term is rejected

Term 1: A is leader
Partition...
Term 2: B elected leader (has majority)
A tries to lead with term 1
All nodes reject A (term 1 < term 2)
```

### Redis Sentinel

**How Redis Sentinel handles split brain**:

1. **Quorum for failover**: Configurable quorum of Sentinels must agree
2. **min-replicas-to-write**: Master rejects writes without enough replicas

```
# redis.conf
min-replicas-to-write 1
min-replicas-max-lag 10

# Master only accepts writes if:
# - At least 1 replica is connected
# - Replica lag is < 10 seconds

# During partition:
# - Isolated master has no replicas
# - Master rejects all writes
# - No split brain data corruption
```

### Kubernetes

**How Kubernetes handles split brain**:

1. **etcd quorum**: Control plane requires etcd majority
2. **Node lease**: Nodes must renew lease to be considered healthy
3. **Pod eviction**: Pods on unreachable nodes are evicted

```yaml
# Node lease renewal
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: node-lease-worker-1
spec:
  holderIdentity: worker-1
  leaseDurationSeconds: 40
  renewTime: "2024-01-15T10:30:00Z"
  
# If node can't renew lease:
# - Node marked NotReady after 40s
# - Pods evicted after 5 minutes
# - Pods rescheduled on healthy nodes
```

### Production War Stories

**Jepsen Testing**: Kyle Kingsbury's Jepsen tests have found split brain vulnerabilities in many databases:
- MongoDB (pre-3.4): Could elect two primaries
- Elasticsearch: Split brain under network partition
- RabbitMQ: Message loss during partition

**GitHub's MySQL Split Brain (2012)**:
- Network partition between MySQL masters
- Both accepted writes
- Required manual data reconciliation
- Led to adoption of better failover mechanisms

---

## 6️⃣ How to Implement or Apply It

### Implementation: Quorum-Based Leader Election

```java
package com.example.splitbrain;

import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

/**
 * Quorum-based leader election that prevents split brain.
 * 
 * Key principle: A node can only be leader if it can reach
 * a majority of nodes in the cluster.
 */
public class QuorumLeaderElection {
    
    private final String nodeId;
    private final List<String> clusterNodes;
    private final int quorumSize;
    private final ClusterCommunication comm;
    
    private volatile String currentLeader;
    private volatile long currentTerm;
    private volatile LeaderState state;
    
    private final ScheduledExecutorService scheduler;
    private final AtomicLong lastLeaderContact;
    
    public QuorumLeaderElection(String nodeId, List<String> clusterNodes,
                                ClusterCommunication comm) {
        this.nodeId = nodeId;
        this.clusterNodes = clusterNodes;
        this.quorumSize = (clusterNodes.size() / 2) + 1;  // Majority
        this.comm = comm;
        this.state = LeaderState.FOLLOWER;
        this.currentTerm = 0;
        this.scheduler = Executors.newScheduledThreadPool(2);
        this.lastLeaderContact = new AtomicLong(System.currentTimeMillis());
    }
    
    /**
     * Start the leader election process.
     */
    public void start() {
        // Start election timeout checker
        scheduler.scheduleAtFixedRate(
            this::checkElectionTimeout,
            1000, 1000, TimeUnit.MILLISECONDS);
        
        // If leader, send heartbeats
        scheduler.scheduleAtFixedRate(
            this::sendHeartbeatsIfLeader,
            500, 500, TimeUnit.MILLISECONDS);
    }
    
    /**
     * Check if we should start an election.
     */
    private void checkElectionTimeout() {
        if (state == LeaderState.LEADER) {
            return;  // Leaders don't timeout
        }
        
        long timeSinceLeader = System.currentTimeMillis() - lastLeaderContact.get();
        if (timeSinceLeader > 3000) {  // 3 second timeout
            startElection();
        }
    }
    
    /**
     * Start a leader election.
     * 
     * Only become leader if we get votes from a quorum.
     */
    private synchronized void startElection() {
        currentTerm++;
        state = LeaderState.CANDIDATE;
        
        System.out.println(nodeId + ": Starting election for term " + currentTerm);
        
        // Vote for self
        Set<String> votes = new HashSet<>();
        votes.add(nodeId);
        
        // Request votes from all other nodes
        for (String node : clusterNodes) {
            if (!node.equals(nodeId)) {
                try {
                    VoteResponse response = comm.requestVote(node, currentTerm, nodeId);
                    if (response.voteGranted && response.term == currentTerm) {
                        votes.add(node);
                    } else if (response.term > currentTerm) {
                        // Someone has higher term, step down
                        currentTerm = response.term;
                        state = LeaderState.FOLLOWER;
                        return;
                    }
                } catch (Exception e) {
                    // Node unreachable, don't count vote
                    System.out.println(nodeId + ": Cannot reach " + node);
                }
            }
        }
        
        // Check if we have quorum
        if (votes.size() >= quorumSize) {
            becomeLeader();
        } else {
            System.out.println(nodeId + ": Election failed, only got " + 
                votes.size() + " votes, need " + quorumSize);
            state = LeaderState.FOLLOWER;
        }
    }
    
    /**
     * Become the leader.
     */
    private void becomeLeader() {
        state = LeaderState.LEADER;
        currentLeader = nodeId;
        System.out.println(nodeId + ": Became leader for term " + currentTerm);
        
        // Immediately send heartbeats
        sendHeartbeatsIfLeader();
    }
    
    /**
     * Send heartbeats to maintain leadership.
     * 
     * Critical: If we can't reach quorum, we must step down.
     */
    private void sendHeartbeatsIfLeader() {
        if (state != LeaderState.LEADER) {
            return;
        }
        
        int reachable = 1;  // Count self
        
        for (String node : clusterNodes) {
            if (!node.equals(nodeId)) {
                try {
                    HeartbeatResponse response = comm.sendHeartbeat(
                        node, currentTerm, nodeId);
                    
                    if (response.success) {
                        reachable++;
                    } else if (response.term > currentTerm) {
                        // Higher term exists, step down
                        stepDown(response.term);
                        return;
                    }
                } catch (Exception e) {
                    // Node unreachable
                }
            }
        }
        
        // CRITICAL: Step down if we can't reach quorum
        if (reachable < quorumSize) {
            System.out.println(nodeId + ": Lost quorum (only " + reachable + 
                " reachable), stepping down");
            state = LeaderState.FOLLOWER;
            currentLeader = null;
        }
    }
    
    /**
     * Handle incoming heartbeat from leader.
     */
    public HeartbeatResponse handleHeartbeat(long term, String leaderId) {
        if (term < currentTerm) {
            return new HeartbeatResponse(false, currentTerm);
        }
        
        if (term > currentTerm) {
            stepDown(term);
        }
        
        currentLeader = leaderId;
        lastLeaderContact.set(System.currentTimeMillis());
        
        return new HeartbeatResponse(true, currentTerm);
    }
    
    /**
     * Handle vote request.
     */
    public VoteResponse handleVoteRequest(long term, String candidateId) {
        if (term < currentTerm) {
            return new VoteResponse(false, currentTerm);
        }
        
        if (term > currentTerm) {
            stepDown(term);
        }
        
        // Simple voting: grant vote if we haven't voted this term
        // In real Raft, also check log completeness
        return new VoteResponse(true, currentTerm);
    }
    
    private void stepDown(long newTerm) {
        currentTerm = newTerm;
        state = LeaderState.FOLLOWER;
        currentLeader = null;
    }
    
    /**
     * Check if this node is the leader.
     */
    public boolean isLeader() {
        return state == LeaderState.LEADER;
    }
    
    /**
     * Get current leader (may be null during election).
     */
    public String getLeader() {
        return currentLeader;
    }
}

enum LeaderState {
    FOLLOWER, CANDIDATE, LEADER
}

record VoteResponse(boolean voteGranted, long term) {}
record HeartbeatResponse(boolean success, long term) {}

interface ClusterCommunication {
    VoteResponse requestVote(String node, long term, String candidateId);
    HeartbeatResponse sendHeartbeat(String node, long term, String leaderId);
}
```

### Implementation: Fencing Token

```java
package com.example.splitbrain;

import java.util.concurrent.atomic.AtomicLong;

/**
 * Fencing token implementation for preventing split brain writes.
 * 
 * Every leader gets a unique, monotonically increasing token.
 * Storage rejects writes with stale tokens.
 */
public class FencingTokenManager {
    
    private final AtomicLong currentToken;
    private final DistributedLock lock;
    
    public FencingTokenManager(DistributedLock lock) {
        this.lock = lock;
        this.currentToken = new AtomicLong(0);
    }
    
    /**
     * Acquire leadership and get a fencing token.
     * 
     * @return Fencing token, or -1 if failed to acquire
     */
    public long acquireLeadership() {
        if (lock.tryAcquire()) {
            long token = currentToken.incrementAndGet();
            System.out.println("Acquired leadership with token " + token);
            return token;
        }
        return -1;
    }
    
    /**
     * Release leadership.
     */
    public void releaseLeadership() {
        lock.release();
    }
    
    /**
     * Get current token (for validation).
     */
    public long getCurrentToken() {
        return currentToken.get();
    }
}

/**
 * Storage that validates fencing tokens.
 */
public class FencedStorage {
    
    private final Map<String, String> data;
    private volatile long acceptedToken;
    
    public FencedStorage() {
        this.data = new ConcurrentHashMap<>();
        this.acceptedToken = 0;
    }
    
    /**
     * Write with fencing token validation.
     * 
     * Rejects writes from old leaders (stale tokens).
     */
    public synchronized void write(long token, String key, String value) {
        if (token < acceptedToken) {
            throw new FencedOffException(
                "Stale token " + token + ", current is " + acceptedToken);
        }
        
        acceptedToken = token;
        data.put(key, value);
        System.out.println("Accepted write with token " + token + 
            ": " + key + "=" + value);
    }
    
    /**
     * Read (no token required).
     */
    public String read(String key) {
        return data.get(key);
    }
    
    /**
     * Get current accepted token.
     */
    public long getAcceptedToken() {
        return acceptedToken;
    }
}

class FencedOffException extends RuntimeException {
    public FencedOffException(String message) {
        super(message);
    }
}
```

### Implementation: Leader Lease

```java
package com.example.splitbrain;

import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

/**
 * Leader lease implementation.
 * 
 * Leader must continuously renew lease.
 * If lease expires, leader must stop acting as leader.
 */
public class LeaderLease {
    
    private final String nodeId;
    private final long leaseDurationMs;
    private final long renewIntervalMs;
    private final long clockDriftBufferMs;
    
    private final QuorumLeaderElection election;
    private final ScheduledExecutorService scheduler;
    
    private volatile long leaseExpiry;
    private volatile boolean isLeaseHolder;
    
    public LeaderLease(String nodeId, QuorumLeaderElection election,
                      long leaseDurationMs, long renewIntervalMs,
                      long clockDriftBufferMs) {
        this.nodeId = nodeId;
        this.election = election;
        this.leaseDurationMs = leaseDurationMs;
        this.renewIntervalMs = renewIntervalMs;
        this.clockDriftBufferMs = clockDriftBufferMs;
        this.scheduler = Executors.newScheduledThreadPool(1);
        this.leaseExpiry = 0;
        this.isLeaseHolder = false;
    }
    
    /**
     * Start lease management.
     */
    public void start() {
        scheduler.scheduleAtFixedRate(
            this::renewLeaseIfLeader,
            0, renewIntervalMs, TimeUnit.MILLISECONDS);
    }
    
    /**
     * Try to acquire or renew the lease.
     */
    private void renewLeaseIfLeader() {
        if (!election.isLeader()) {
            isLeaseHolder = false;
            return;
        }
        
        // Try to renew lease with quorum
        if (tryRenewWithQuorum()) {
            leaseExpiry = System.currentTimeMillis() + leaseDurationMs;
            isLeaseHolder = true;
            System.out.println(nodeId + ": Lease renewed until " + leaseExpiry);
        } else {
            // Failed to renew - must stop acting as leader
            System.out.println(nodeId + ": Failed to renew lease, stepping down");
            isLeaseHolder = false;
        }
    }
    
    /**
     * Check if we can act as leader.
     * 
     * CRITICAL: Must have valid lease with buffer for clock drift.
     */
    public boolean canActAsLeader() {
        if (!isLeaseHolder) {
            return false;
        }
        
        // Account for clock drift
        long safeExpiry = leaseExpiry - clockDriftBufferMs;
        return System.currentTimeMillis() < safeExpiry;
    }
    
    /**
     * Execute operation only if we're the valid leader.
     */
    public <T> T executeAsLeader(Callable<T> operation) throws Exception {
        if (!canActAsLeader()) {
            throw new NotLeaderException("Not the lease holder or lease expired");
        }
        
        // Double-check after acquiring work
        T result = operation.call();
        
        // Verify we still have lease after operation
        if (!canActAsLeader()) {
            throw new LeaseExpiredException(
                "Lease expired during operation, result may be invalid");
        }
        
        return result;
    }
    
    private boolean tryRenewWithQuorum() {
        // In real implementation, this would contact quorum
        // and get acknowledgment of lease renewal
        return election.isLeader();  // Simplified
    }
}

class NotLeaderException extends RuntimeException {
    public NotLeaderException(String message) {
        super(message);
    }
}

class LeaseExpiredException extends RuntimeException {
    public LeaseExpiredException(String message) {
        super(message);
    }
}
```

### Implementation: STONITH

```java
package com.example.splitbrain;

/**
 * STONITH (Shoot The Other Node In The Head) implementation.
 * 
 * Uses out-of-band management to forcibly terminate nodes
 * before taking over leadership.
 */
public class StonithManager {
    
    private final String nodeId;
    private final Map<String, NodeManagement> managementInterfaces;
    
    public StonithManager(String nodeId, 
                         Map<String, NodeManagement> managementInterfaces) {
        this.nodeId = nodeId;
        this.managementInterfaces = managementInterfaces;
    }
    
    /**
     * STONITH a node before taking over.
     * 
     * @param targetNode Node to terminate
     * @return true if node is confirmed dead
     */
    public boolean stonith(String targetNode) {
        NodeManagement mgmt = managementInterfaces.get(targetNode);
        if (mgmt == null) {
            System.err.println("No management interface for " + targetNode);
            return false;
        }
        
        System.out.println(nodeId + ": STONITHing " + targetNode);
        
        // Step 1: Send power off command
        boolean powerOffSent = mgmt.powerOff();
        if (!powerOffSent) {
            System.err.println("Failed to send power off to " + targetNode);
            return false;
        }
        
        // Step 2: Wait for confirmation
        try {
            Thread.sleep(5000);  // Wait for power off to take effect
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
        
        // Step 3: Verify node is dead
        boolean isDead = mgmt.isPoweredOff();
        if (isDead) {
            System.out.println(nodeId + ": Confirmed " + targetNode + " is dead");
        } else {
            System.err.println(nodeId + ": " + targetNode + " still alive!");
        }
        
        return isDead;
    }
    
    /**
     * Safe failover with STONITH.
     */
    public boolean safeFailover(String oldLeader) {
        // Step 1: STONITH the old leader
        if (!stonith(oldLeader)) {
            System.err.println("STONITH failed, cannot safely failover");
            return false;
        }
        
        // Step 2: Now safe to become leader
        System.out.println(nodeId + ": Safe to become leader");
        return true;
    }
}

/**
 * Interface for out-of-band node management.
 * Implementations: IPMI, iLO, DRAC, cloud APIs
 */
interface NodeManagement {
    boolean powerOff();
    boolean powerOn();
    boolean isPoweredOff();
    boolean reset();
}

/**
 * IPMI implementation of node management.
 */
class IpmiNodeManagement implements NodeManagement {
    private final String ipmiAddress;
    private final String username;
    private final String password;
    
    public IpmiNodeManagement(String ipmiAddress, String username, String password) {
        this.ipmiAddress = ipmiAddress;
        this.username = username;
        this.password = password;
    }
    
    @Override
    public boolean powerOff() {
        // Execute: ipmitool -H <address> -U <user> -P <pass> power off
        try {
            ProcessBuilder pb = new ProcessBuilder(
                "ipmitool", "-H", ipmiAddress, 
                "-U", username, "-P", password,
                "power", "off");
            Process p = pb.start();
            return p.waitFor() == 0;
        } catch (Exception e) {
            return false;
        }
    }
    
    @Override
    public boolean isPoweredOff() {
        // Execute: ipmitool -H <address> -U <user> -P <pass> power status
        try {
            ProcessBuilder pb = new ProcessBuilder(
                "ipmitool", "-H", ipmiAddress,
                "-U", username, "-P", password,
                "power", "status");
            Process p = pb.start();
            // Parse output to check power state
            return true;  // Simplified
        } catch (Exception e) {
            return false;
        }
    }
    
    @Override
    public boolean powerOn() {
        // Similar to powerOff
        return true;
    }
    
    @Override
    public boolean reset() {
        // Similar to powerOff with "reset" command
        return true;
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Tradeoffs

| Strategy | Pros | Cons |
|----------|------|------|
| Quorum | Mathematically prevents split brain | Requires majority, reduced availability |
| Fencing tokens | Simple, works with any storage | Requires token-aware storage |
| STONITH | Guaranteed single leader | Aggressive, requires out-of-band network |
| Leader lease | Time-bounded authority | Clock drift issues |

### Common Mistakes

**1. Not Accounting for Clock Drift**
```java
// BAD: Assumes perfect clocks
boolean isLeaseValid() {
    return System.currentTimeMillis() < leaseExpiry;
}

// GOOD: Account for clock drift
boolean isLeaseValid() {
    long buffer = 1000;  // 1 second buffer
    return System.currentTimeMillis() < leaseExpiry - buffer;
}
```

**2. Ignoring GC Pauses**
```java
// BAD: Check lease, then do work
if (isLeader()) {
    // GC pause happens here for 30 seconds
    // Lease expires during pause
    // But we continue as if we're leader
    doLeaderWork();
}

// GOOD: Check lease before AND after
if (isLeader()) {
    Result result = doLeaderWork();
    if (!isLeader()) {
        // Lease expired during work
        throw new LeaseExpiredException();
    }
    return result;
}
```

**3. Quorum Misconfiguration**
```java
// BAD: Even number of nodes
int nodes = 4;
int quorum = 2;  // Two partitions of 2 can both have "quorum"

// GOOD: Odd number of nodes
int nodes = 5;
int quorum = 3;  // Only one partition can have majority
```

**4. Not Fencing Old Leaders**
```java
// BAD: New leader starts without fencing old
void becomeLeader() {
    isLeader = true;
    startAcceptingWrites();  // Old leader might still be writing!
}

// GOOD: Fence before becoming leader
void becomeLeader() {
    long newToken = fencingTokenManager.acquireLeadership();
    storage.setMinimumToken(newToken);  // Old tokens rejected
    isLeader = true;
    startAcceptingWrites();
}
```

**5. Asymmetric Partition Handling**
```java
// BAD: Only check if we can send
if (canSendTo(otherNode)) {
    // But otherNode might not be able to send to us!
    assumeHealthy(otherNode);
}

// GOOD: Require bidirectional communication
if (canSendTo(otherNode) && receivedRecentHeartbeat(otherNode)) {
    assumeHealthy(otherNode);
}
```

### Performance Gotchas

**Quorum Latency**: Every write requires quorum acknowledgment. Latency = max(quorum node latencies).

**Lease Renewal Overhead**: Frequent lease renewal adds network traffic and latency.

**STONITH Delay**: STONITH operations take seconds, increasing failover time.

---

## 8️⃣ When NOT to Use This

### When Split Brain Prevention Is Overkill

**Single Node Systems**
- No other nodes to conflict with
- Split brain impossible
- Just handle single node failures

**Stateless Services**
- No shared state to corrupt
- Multiple instances are fine
- Load balancer handles distribution

**Idempotent Operations**
- Duplicate operations are safe
- No need to prevent dual leaders
- Eventually consistent is acceptable

**Read-Only Systems**
- No writes to conflict
- Multiple readers are fine
- Focus on availability over consistency

### When to Accept Split Brain Risk

**Availability Over Consistency**
- User experience more important than consistency
- Can reconcile conflicts later
- Example: Shopping cart (merge on conflict)

**Conflict Resolution Possible**
- Last-write-wins acceptable
- CRDTs for automatic merge
- Manual reconciliation feasible

---

## 9️⃣ Comparison with Alternatives

### Prevention Strategies Comparison

| Strategy | Complexity | Availability | Safety | Use Case |
|----------|------------|--------------|--------|----------|
| Quorum | Medium | Lower | High | Databases, consensus |
| Fencing | Low | High | High | Storage systems |
| STONITH | High | Medium | Very High | Critical systems |
| Lease | Medium | High | Medium | Distributed locks |

### Consensus Protocols

| Protocol | Split Brain Safe | Availability | Complexity |
|----------|-----------------|--------------|------------|
| Raft | Yes | Majority required | Medium |
| Paxos | Yes | Majority required | High |
| ZAB | Yes | Majority required | Medium |
| 2PC | No (blocking) | Low | Low |

---

## 🔟 Interview Follow-up Questions WITH Answers

### L4 (Entry Level) Questions

**Q1: What is the split brain problem?**

**A**: Split brain occurs when a distributed system's nodes get divided into groups that can't communicate (network partition), and multiple groups independently believe they are the authoritative source.

**Example**:
- 3-node database cluster with Node A as leader
- Network partition isolates A from B and C
- A continues accepting writes (thinks it's still leader)
- B and C elect B as new leader, B accepts writes
- Now two leaders accepting conflicting writes
- When partition heals, data is inconsistent

**Why it's dangerous**:
- Data corruption
- Lost transactions
- Inconsistent reads
- Application errors

---

**Q2: How does quorum prevent split brain?**

**A**: Quorum requires a majority of nodes to agree before making any decision. Since two disjoint groups cannot both have majority, only one partition can make progress.

**Example with 5 nodes**:
```
Quorum = 3 (majority of 5)

Partition: [A, B] | [C, D, E]
           2 nodes | 3 nodes

[A, B]: Only 2 nodes, cannot form quorum
        Cannot elect leader, cannot accept writes
        
[C, D, E]: 3 nodes = quorum
           Can elect leader, can accept writes
           
Result: Only one partition operates, no split brain
```

**Key insight**: Any two majorities must overlap by at least one node, so they can't make conflicting decisions.

---

### L5 (Mid Level) Questions

**Q3: Explain fencing tokens and how they prevent split brain.**

**A**: Fencing tokens are monotonically increasing numbers assigned to leaders. Storage systems reject operations from leaders with stale (lower) tokens.

**How it works**:

1. **Token assignment**:
```
Leader A elected → Token 1
Leader A's writes tagged with Token 1
```

2. **Partition and new election**:
```
Partition isolates A
B elected as new leader → Token 2
Storage updates minimum accepted token to 2
```

3. **Old leader fenced**:
```
A tries to write with Token 1
Storage rejects: Token 1 < minimum Token 2
A is "fenced off"
```

**Why it works**:
- Tokens are monotonically increasing
- Storage only accepts current or higher tokens
- Old leaders automatically rejected
- No coordination needed at write time

**Implementation considerations**:
- Token must be persisted durably
- Storage must check token atomically with write
- Works with any storage that supports conditional writes

---

**Q4: What are the challenges with leader leases and how do you address them?**

**A**: Leader leases grant time-bounded authority. The leader can act only while the lease is valid. Challenges include:

**Challenge 1: Clock Drift**
```
Leader thinks: Lease expires at 10:00:00
Follower thinks: Lease expires at 09:59:58 (clock 2s behind)

At 09:59:59 (leader's clock):
- Leader thinks lease valid (1 second left)
- Follower thinks lease expired (1 second ago)
- Follower might elect new leader
- Two leaders!
```

**Solution**: Conservative buffer
```java
boolean canActAsLeader() {
    long buffer = maxClockDrift + safetyMargin;  // e.g., 5 seconds
    return now() < leaseExpiry - buffer;
}
```

**Challenge 2: GC Pauses**
```
Leader checks lease: valid
GC pause: 30 seconds
Lease expires during pause
Leader continues after pause, thinks it's still leader
```

**Solution**: Check before AND after operations
```java
void doLeaderOperation() {
    checkLease();  // Before
    performOperation();
    checkLease();  // After - throw if expired
}
```

**Challenge 3: Network Delays**
```
Leader sends heartbeat at T
Network delays heartbeat by 10 seconds
Followers think leader dead at T+5
New leader elected
Original heartbeat arrives at T+10
Confusion about who's leader
```

**Solution**: Include timestamp in heartbeat, reject stale heartbeats
```java
void handleHeartbeat(long timestamp, long term) {
    if (now() - timestamp > maxHeartbeatAge) {
        reject("Stale heartbeat");
    }
    // Process heartbeat
}
```

---

### L6 (Senior Level) Questions

**Q5: Design a split-brain-safe distributed lock service.**

**A**: A distributed lock service must ensure only one client holds a lock at a time, even during partitions.

**Architecture**:
```
┌─────────────────────────────────────────────────────────────┐
│                    Lock Service Cluster                      │
│                                                              │
│    ┌─────────┐    ┌─────────┐    ┌─────────┐               │
│    │ Node 1  │────│ Node 2  │────│ Node 3  │               │
│    │ (Leader)│    │(Follower)│    │(Follower)│              │
│    └─────────┘    └─────────┘    └─────────┘               │
│         │              │              │                     │
│         └──────────────┴──────────────┘                     │
│                    Raft Consensus                            │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│                       Clients                                │
│    ┌────────┐    ┌────────┐    ┌────────┐                  │
│    │Client A│    │Client B│    │Client C│                  │
│    └────────┘    └────────┘    └────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

**Key Design Decisions**:

**1. Raft-based consensus**
```java
class LockService {
    private final RaftConsensus raft;
    
    public LockResult acquireLock(String lockId, String clientId, long ttlMs) {
        // Create lock command
        LockCommand cmd = new LockCommand(ACQUIRE, lockId, clientId, ttlMs);
        
        // Replicate via Raft (requires quorum)
        RaftResult result = raft.replicate(cmd);
        
        if (!result.isCommitted()) {
            return LockResult.failed("Not committed");
        }
        
        return LockResult.success(result.getFencingToken());
    }
}
```

**2. Fencing tokens for lock holders**
```java
class Lock {
    String lockId;
    String holderId;
    long fencingToken;  // Monotonically increasing
    long expiryTime;
    
    boolean isHeldBy(String clientId, long token) {
        return holderId.equals(clientId) 
            && token == fencingToken
            && System.currentTimeMillis() < expiryTime;
    }
}
```

**3. TTL-based expiry**
```java
// Client must renew lock before TTL expires
class LockClient {
    void renewLock(String lockId, long token) {
        // Must use same fencing token
        lockService.renew(lockId, clientId, token);
    }
    
    // Background renewal
    void startRenewal(String lockId, long token, long ttlMs) {
        scheduler.scheduleAtFixedRate(() -> {
            try {
                renewLock(lockId, token);
            } catch (Exception e) {
                // Lost lock
                onLockLost();
            }
        }, ttlMs / 3, ttlMs / 3, TimeUnit.MILLISECONDS);
    }
}
```

**4. Client-side safety**
```java
class SafeLockClient {
    void executeWithLock(String lockId, Runnable action) {
        LockResult lock = lockService.acquire(lockId, clientId, ttlMs);
        if (!lock.isSuccess()) {
            throw new LockFailedException();
        }
        
        try {
            // Pass fencing token to protected resource
            protectedResource.setFencingToken(lock.getToken());
            action.run();
        } finally {
            lockService.release(lockId, lock.getToken());
        }
    }
}
```

**5. Split brain handling**
```
Scenario: Partition isolates leader

Before partition:
- Node 1 is leader
- Client A holds lock with token 5

During partition:
- Node 1 isolated (minority)
- Nodes 2, 3 elect Node 2 as leader
- Client B acquires lock with token 6
- Node 1 cannot renew Client A's lock (no quorum)
- Client A's lock expires

After partition heals:
- Node 1 rejoins as follower
- Only Client B holds lock (token 6)
- Client A's operations rejected (token 5 < 6)
```

---

**Q6: How would you handle split brain in a multi-datacenter deployment?**

**A**: Multi-datacenter deployments have unique challenges: higher latency, datacenter-level failures, and the need for local availability.

**Architecture Options**:

**Option 1: Single Raft cluster across DCs**
```
DC1 (Primary)        DC2 (Secondary)      DC3 (Witness)
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Node 1     │     │  Node 3     │     │  Node 5     │
│  Node 2     │     │  Node 4     │     │ (no data)   │
└─────────────┘     └─────────────┘     └─────────────┘
      │                   │                   │
      └───────────────────┴───────────────────┘
              Raft consensus (quorum = 3)
```

**Pros**: Strong consistency, automatic failover
**Cons**: High latency (cross-DC for every write), requires 3 DCs

**Option 2: Leader per DC with async replication**
```
DC1                      DC2
┌─────────────┐         ┌─────────────┐
│  Leader 1   │◄───────►│  Leader 2   │
│  (Region A) │  async  │  (Region B) │
└─────────────┘         └─────────────┘

Each DC handles its own region
Async replication for disaster recovery
Accept eventual consistency
```

**Pros**: Low latency, DC-local availability
**Cons**: Eventual consistency, conflict resolution needed

**Option 3: Hierarchical consensus**
```
Global Layer (across DCs):
┌─────────────────────────────────────────────┐
│           Global Raft (metadata)            │
│  DC1-rep ◄──────► DC2-rep ◄──────► DC3-rep │
└─────────────────────────────────────────────┘
                      │
        ┌─────────────┴─────────────┐
        ▼                           ▼
Local Layer (within DC):     Local Layer (within DC):
┌─────────────────┐         ┌─────────────────┐
│ DC1 Local Raft  │         │ DC2 Local Raft  │
│ (data)          │         │ (data)          │
└─────────────────┘         └─────────────────┘
```

**Pros**: Local writes fast, global consistency for metadata
**Cons**: Complex, different consistency for different data

**Split Brain Prevention in Multi-DC**:

**1. Witness/Arbiter DC**
```
DC1 and DC2 can't communicate
Witness DC breaks the tie:
- If DC1 can reach Witness: DC1 has quorum
- If DC2 can reach Witness: DC2 has quorum
- If neither can reach Witness: both stop (safe)
```

**2. Geo-aware quorum**
```java
class GeoAwareQuorum {
    // Require nodes from multiple DCs
    boolean hasQuorum(Set<Node> reachable) {
        Map<String, Long> dcCounts = reachable.stream()
            .collect(groupingBy(Node::getDc, counting()));
        
        // Need majority of DCs AND majority of total nodes
        int dcsReached = dcCounts.size();
        int nodesReached = reachable.size();
        
        return dcsReached >= (totalDcs / 2 + 1) 
            && nodesReached >= (totalNodes / 2 + 1);
    }
}
```

**3. Conflict-free data types (CRDTs)**
```java
// For data that can tolerate conflicts
class GCounter {  // Grow-only counter
    Map<String, Long> counts;  // DC -> count
    
    void increment(String dc) {
        counts.merge(dc, 1L, Long::sum);
    }
    
    GCounter merge(GCounter other) {
        // Take max from each DC - no conflicts
        GCounter merged = new GCounter();
        for (String dc : union(counts.keySet(), other.counts.keySet())) {
            merged.counts.put(dc, 
                Math.max(counts.getOrDefault(dc, 0L),
                        other.counts.getOrDefault(dc, 0L)));
        }
        return merged;
    }
    
    long value() {
        return counts.values().stream().mapToLong(Long::longValue).sum();
    }
}
```

---

## 1️⃣1️⃣ One Clean Mental Summary

Split brain occurs when network partitions cause multiple nodes in a distributed system to independently believe they are the authoritative leader, leading to conflicting operations and data corruption.

The fundamental challenge is that nodes cannot distinguish between "the other node is dead" and "the network between us is broken." Both look the same from each node's perspective.

**Prevention strategies** all enforce a single principle: **only one partition can have authority at a time**.
- **Quorum**: Majority required, only one partition can have majority
- **Fencing tokens**: Monotonic tokens reject stale leaders
- **STONITH**: Forcibly kill other node before taking over
- **Leader lease**: Time-bounded authority that expires

In production, use **Raft/Paxos-based systems** (etcd, ZooKeeper) for coordination, **odd number of nodes** for clear majorities, and **fencing tokens** for storage access. Always account for **clock drift** and **GC pauses** when implementing leases.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│                    SPLIT BRAIN                               │
├─────────────────────────────────────────────────────────────┤
│ Cause              │ Description                             │
├─────────────────────────────────────────────────────────────┤
│ Network partition  │ Nodes can't communicate                 │
│ Node isolation     │ Single node cut off                     │
│ Asymmetric network │ One-way communication                   │
│ GC pause           │ Node appears dead but isn't             │
├─────────────────────────────────────────────────────────────┤
│                    PREVENTION                                │
├─────────────────────────────────────────────────────────────┤
│ Quorum             │ Majority required for decisions         │
│ Fencing tokens     │ Monotonic tokens reject old leaders     │
│ STONITH            │ Kill other node before taking over      │
│ Leader lease       │ Time-bounded authority                  │
├─────────────────────────────────────────────────────────────┤
│                    BEST PRACTICES                            │
├─────────────────────────────────────────────────────────────┤
│ Odd node count     │ 3, 5, 7 nodes for clear majority       │
│ Clock drift buffer │ Account for clock differences           │
│ GC awareness       │ Check authority before AND after ops    │
│ Fencing at storage │ Storage rejects stale tokens            │
├─────────────────────────────────────────────────────────────┤
│                    QUORUM FORMULA                            │
├─────────────────────────────────────────────────────────────┤
│ Quorum size        │ (n / 2) + 1 where n = total nodes      │
│ 3 nodes            │ Quorum = 2                              │
│ 5 nodes            │ Quorum = 3                              │
│ 7 nodes            │ Quorum = 4                              │
└─────────────────────────────────────────────────────────────┘
```

