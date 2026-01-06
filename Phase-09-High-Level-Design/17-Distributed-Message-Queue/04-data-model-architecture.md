# Distributed Message Queue - Data Model & Architecture

## 1. High-Level Architecture

### System Overview

```mermaid
flowchart TB
    Producers["Producers<br/>Producer 1, Producer 2, Producer N"]
    Producers --> MetadataCache
    MetadataCache["Metadata Cache<br/>Topic/Partition info"]
    MetadataCache --> Broker1
    MetadataCache --> Broker2
    MetadataCache --> Broker3
    MetadataCache --> BrokerN
    Broker1["Broker 1"]
    Broker2["Broker 2"]
    Broker3["Broker 3"]
    BrokerN["Broker N"]
    Broker1 <-->|"Replication"| Broker2
    Broker2 <-->|"Replication"| Broker3
    Broker3 <-->|"Replication"| BrokerN
    Broker1 --> Disk1
    Broker2 --> Disk2
    Broker3 --> Disk3
    BrokerN --> DiskN
    Disk1["Disk (Logs)"]
    Disk2["Disk (Logs)"]
    Disk3["Disk (Logs)"]
    DiskN["Disk (Logs)"]
    Broker1 --> GroupA
    Broker2 --> GroupA
    Broker3 --> GroupB
    BrokerN --> GroupB
    GroupA["Group A"]
    GroupB["Group B"]
    GroupA --> Consumers
    GroupB --> Consumers
    Consumers["Consumers"]
    ControllerLeader["Controller (Leader)"]
    ControllerFollower1["Controller (Follower)"]
    ControllerFollower2["Controller (Follower)"]
    ControllerLeader <-->|"Raft"| ControllerFollower1
    ControllerLeader <-->|"Raft"| ControllerFollower2
    ControllerFollower1 <-->|"Raft"| ControllerFollower2
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    Distributed Message Queue Architecture                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                   │
│  Producers                                                                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                                       │
│  │Producer 1│  │Producer 2│  │Producer N│                                       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                                       │
│       │             │             │                                              │
│       └─────────────┴─────────────┘                                              │
│                     │                                                            │
│              ┌──────▼──────┐                                                     │
│              │   Metadata  │ ◄─── Topic/Partition info                          │
│              │   Cache     │                                                     │
│              └──────┬──────┘                                                     │
│                     │                                                            │
│    ┌────────────────┼────────────────┬────────────────┐                         │
│    │                │                │                │                          │
│    ▼                ▼                ▼                ▼                          │
│ ┌──────┐        ┌──────┐        ┌──────┐        ┌──────┐                        │
│ │Broker│        │Broker│        │Broker│        │Broker│                        │
│ │  1   │◄──────►│  2   │◄──────►│  3   │◄──────►│  N   │                        │
│ └──┬───┘        └──┬───┘        └──┬───┘        └──┬───┘                        │
│    │               │               │               │                            │
│    │  Replication  │               │               │                            │
│    │               │               │               │                            │
│ ┌──┴───┐        ┌──┴───┐        ┌──┴───┐        ┌──┴───┐                        │
│ │ Disk │        │ Disk │        │ Disk │        │ Disk │                        │
│ │(Logs)│        │(Logs)│        │(Logs)│        │(Logs)│                        │
│ └──────┘        └──────┘        └──────┘        └──────┘                        │
│                                                                                   │
│                     │                                                            │
│       ┌─────────────┴─────────────┐                                             │
│       │                           │                                              │
│    ┌──▼───┐                    ┌──▼───┐                                         │
│    │Group │                    │Group │                                         │
│    │  A   │                    │  B   │                                         │
│    └──────┘                    └──────┘                                         │
│  Consumers                                                                       │
│                                                                                   │
│  Controller Cluster (Raft-based)                                                 │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                                 │
│  │ Controller │  │ Controller │  │ Controller │                                 │
│  │   (Leader) │  │ (Follower) │  │ (Follower) │                                 │
│  └────────────┘  └────────────┘  └────────────┘                                 │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

</details>
```

---

## 2. Core Components

### 2.1 Broker

```
Purpose: Store and serve messages
Responsibilities:
├── Accept produce requests
├── Store messages to disk
├── Serve fetch requests
├── Replicate to followers
├── Participate in leader election
└── Report metrics

Key Subsystems:
├── Network Layer (Acceptor, Processor threads)
├── Request Handler (API processing)
├── Log Manager (Segment management)
├── Replica Manager (Replication)
├── Group Coordinator (Consumer groups)
└── Transaction Coordinator (Exactly-once)
```

### 2.2 Controller

```
Purpose: Cluster metadata management
Responsibilities:
├── Topic/partition management
├── Broker registration
├── Leader election
├── Partition reassignment
├── Configuration management
└── ACL management

Implementation:
├── Raft consensus (KRaft mode)
├── Replicated state machine
├── Metadata log
└── Snapshot for recovery
```

### 2.3 Log Manager

```
Purpose: Manage on-disk message storage
Responsibilities:
├── Segment management
├── Index maintenance
├── Log cleaning/compaction
├── Retention enforcement
└── Recovery after crash

Structure:
├── Active segment (current writes)
├── Closed segments (read-only)
├── Offset index (sparse)
├── Time index
└── Transaction index
```

---

## 3. Data Flow Diagrams

### 3.1 Produce Flow

```mermaid
sequenceDiagram
    participant Producer
    participant Leader as Broker (Leader)
    participant Follower1 as Broker (Follower)
    participant Follower2 as Broker (F2)
    
    Producer->>Leader: 1. Produce (acks=all)
    Leader->>Leader: 2. Write to log<br/>(append to segment)
    Leader->>Leader: 3. Wait for ISR replication
    Follower1->>Leader: 4. Fetch request
    Leader->>Follower1: 5. Send records
    Follower2->>Leader: 6. Fetch request
    Leader->>Follower2: 7. Send records
    Leader->>Leader: 8. All ISR acked
    Leader->>Producer: 9. Produce ACK<br/>(offset assigned)
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│                            Produce Flow                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Producer          Broker (Leader)        Broker (Follower)     Broker (F2) │
│     │                    │                      │                    │       │
│     │  1. Produce        │                      │                    │       │
│     │  (acks=all)        │                      │                    │       │
│     │───────────────────>│                      │                    │       │
│     │                    │                      │                    │       │
│     │                    │  2. Write to log     │                    │       │
│     │                    │  (append to segment) │                    │       │
│     │                    │                      │                    │       │
│     │                    │  3. Wait for ISR     │                    │       │
│     │                    │  replication         │                    │       │
│     │                    │                      │                    │       │
│     │                    │  4. Fetch request    │                    │       │
│     │                    │<─────────────────────│                    │       │
│     │                    │                      │                    │       │
│     │                    │  5. Send records     │                    │       │
│     │                    │─────────────────────>│                    │       │
│     │                    │                      │                    │       │
│     │                    │  6. Fetch request    │                    │       │
│     │                    │<──────────────────────────────────────────│       │
│     │                    │                      │                    │       │
│     │                    │  7. Send records     │                    │       │
│     │                    │──────────────────────────────────────────>│       │
│     │                    │                      │                    │       │
│     │                    │  8. All ISR acked    │                    │       │
│     │                    │                      │                    │       │
│     │  9. Produce ACK    │                      │                    │       │
│     │  (offset assigned) │                      │                    │       │
│     │<───────────────────│                      │                    │       │
│     │                    │                      │                    │       │
└──────────────────────────────────────────────────────────────────────────────┘
```

</details>
```

### 3.2 Consume Flow

```mermaid
sequenceDiagram
    participant Consumer
    participant Coordinator
    participant Broker as Broker (Leader)
    
    Consumer->>Coordinator: 1. FindCoordinator
    Coordinator->>Consumer: 2. Coordinator info
    Consumer->>Coordinator: 3. JoinGroup
    Coordinator->>Consumer: 4. JoinGroup response<br/>(assignment if leader)
    Consumer->>Coordinator: 5. SyncGroup<br/>(leader sends assign)
    Coordinator->>Consumer: 6. SyncGroup response<br/>(partition assignment)
    Consumer->>Broker: 7. Fetch
    Broker->>Consumer: 8. Records
    Consumer->>Coordinator: 9. OffsetCommit
    loop Periodic
        Consumer->>Coordinator: 10. Heartbeat
    end
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│                            Consume Flow                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Consumer              Coordinator              Broker (Leader)              │
│     │                       │                        │                       │
│     │  1. FindCoordinator   │                        │                       │
│     │──────────────────────>│                        │                       │
│     │                       │                        │                       │
│     │  2. Coordinator info  │                        │                       │
│     │<──────────────────────│                        │                       │
│     │                       │                        │                       │
│     │  3. JoinGroup         │                        │                       │
│     │──────────────────────>│                        │                       │
│     │                       │                        │                       │
│     │  4. JoinGroup response│                        │                       │
│     │  (assignment if leader)                        │                       │
│     │<──────────────────────│                        │                       │
│     │                       │                        │                       │
│     │  5. SyncGroup         │                        │                       │
│     │  (leader sends assign)│                        │                       │
│     │──────────────────────>│                        │                       │
│     │                       │                        │                       │
│     │  6. SyncGroup response│                        │                       │
│     │  (partition assignment)                        │                       │
│     │<──────────────────────│                        │                       │
│     │                       │                        │                       │
│     │  7. Fetch             │                        │                       │
│     │─────────────────────────────────────────────-->│                       │
│     │                       │                        │                       │
│     │  8. Records           │                        │                       │
│     │<─────────────────────────────────────────────────│                     │
│     │                       │                        │                       │
│     │  9. OffsetCommit      │                        │                       │
│     │──────────────────────>│                        │                       │
│     │                       │                        │                       │
│     │  10. Heartbeat (periodic)                      │                       │
│     │──────────────────────>│                        │                       │
│     │                       │                        │                       │
└──────────────────────────────────────────────────────────────────────────────┘
```

</details>
```

### 3.3 Replication Flow

```mermaid
sequenceDiagram
    participant Leader
    participant Follower1
    participant Follower2
    
    Note over Leader: High Watermark: 100<br/>LEO: 105
    Note over Follower1: LEO: 95
    Note over Follower2: LEO: 90
    
    Follower1->>Leader: Fetch(offset=95)
    Leader->>Follower1: Records [95-105]<br/>HW: 100
    Follower1->>Follower1: Write to log<br/>LEO: 105
    
    Follower2->>Leader: Fetch(offset=90)
    Leader->>Follower2: Records [90-105]<br/>HW: 100
    Follower2->>Follower2: Write to log<br/>LEO: 105
    
    Note over Leader: All ISR at LEO 105<br/>Advance HW to 105
    Note over Leader,Follower2: Next fetch includes<br/>HW: 105
```

**Key Concepts:**
- LEO (Log End Offset): Last offset written to log
- HW (High Watermark): Last offset replicated to all ISR
- ISR (In-Sync Replicas): Replicas within lag threshold
- Consumers only see up to HW (committed messages)

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│                          Replication Flow                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Leader                  Follower 1               Follower 2                 │
│    │                         │                        │                       │
│    │  High Watermark: 100    │  LEO: 95              │  LEO: 90              │
│    │  LEO: 105               │                        │                       │
│    │                         │                        │                       │
│    │                         │  Fetch(offset=95)     │                       │
│    │<────────────────────────│                        │                       │
│    │                         │                        │                       │
│    │  Records [95-105]       │                        │                       │
│    │  HW: 100                │                        │                       │
│    │────────────────────────>│                        │                       │
│    │                         │                        │                       │
│    │                         │  Write to log         │                       │
│    │                         │  LEO: 105             │                       │
│    │                         │                        │                       │
│    │                         │                        │  Fetch(offset=90)    │
│    │<────────────────────────────────────────────────│                       │
│    │                         │                        │                       │
│    │  Records [90-105]       │                        │                       │
│    │  HW: 100                │                        │                       │
│    │────────────────────────────────────────────────>│                       │
│    │                         │                        │                       │
│    │                         │                        │  Write to log        │
│    │                         │                        │  LEO: 105            │
│    │                         │                        │                       │
│    │  All ISR at LEO 105     │                        │                       │
│    │  Advance HW to 105      │                        │                       │
│    │                         │                        │                       │
│    │  Next fetch includes    │                        │                       │
│    │  HW: 105                │                        │                       │
│    │                         │                        │                       │
└──────────────────────────────────────────────────────────────────────────────┘

Key Concepts:
├── LEO (Log End Offset): Last offset written to log
├── HW (High Watermark): Last offset replicated to all ISR
├── ISR (In-Sync Replicas): Replicas within lag threshold
└── Consumers only see up to HW (committed messages)
```

</details>
```

---

## 4. Log Storage Architecture

### Segment Structure

```mermaid
flowchart TD
    Partition["Partition Directory:<br/>/data/kafka-logs/orders-0/"]
    
    subgraph Segment0["Segment 0 (base offset: 0)"]
        S0Log["00000000000000000000.log<br/>(1 GB, closed)"]
        S0Index["00000000000000000000.index<br/>(10 MB)"]
        S0TimeIndex["00000000000000000000.timeindex<br/>(10 MB)"]
    end
    
    subgraph Segment1["Segment 1 (base offset: 1000000)"]
        S1Log["00000000000001000000.log<br/>(1 GB, closed)"]
        S1Index["00000000000001000000.index"]
        S1TimeIndex["00000000000001000000.timeindex"]
    end
    
    subgraph Segment2["Segment 2 (base offset: 2000000) - ACTIVE"]
        S2Log["00000000000002000000.log<br/>(500 MB, active)"]
        S2Index["00000000000002000000.index"]
        S2TimeIndex["00000000000002000000.timeindex"]
    end
    
    Lookup["Offset Lookup:<br/>1. Binary search in index files to find segment<br/>2. Binary search in segment's index for position<br/>3. Sequential scan from position to find exact offset"]
    
    Partition --> Segment0
    Partition --> Segment1
    Partition --> Segment2
    Segment0 --> Lookup
    Segment1 --> Lookup
    Segment2 --> Lookup
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Log Segment Structure                                 │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Partition Directory: /data/kafka-logs/orders-0/                             │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ Segment 0 (base offset: 0)                                          │     │
│  │ ├── 00000000000000000000.log      (1 GB, closed)                   │     │
│  │ ├── 00000000000000000000.index    (10 MB)                          │     │
│  │ └── 00000000000000000000.timeindex (10 MB)                         │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ Segment 1 (base offset: 1000000)                                    │     │
│  │ ├── 00000000000001000000.log      (1 GB, closed)                   │     │
│  │ ├── 00000000000001000000.index                                     │     │
│  │ └── 00000000000001000000.timeindex                                 │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ Segment 2 (base offset: 2000000) - ACTIVE                          │     │
│  │ ├── 00000000000002000000.log      (500 MB, active)                 │     │
│  │ ├── 00000000000002000000.index                                     │     │
│  │ └── 00000000000002000000.timeindex                                 │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  Offset Lookup:                                                               │
│  1. Binary search in index files to find segment                             │
│  2. Binary search in segment's index for position                            │
│  3. Sequential scan from position to find exact offset                       │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

</details>
```

### Log Compaction

```mermaid
flowchart LR
    subgraph Before["Before Compaction (cleanup.policy=compact)"]
        B0["Offset 0: Key A = v1"]
        B1["Offset 1: Key B = v1"]
        B2["Offset 2: Key A = v2<br/>(← Newer value for key A)"]
        B3["Offset 3: Key C = v1"]
        B4["Offset 4: Key B = v2<br/>(← Newer value for key B)"]
        B5["Offset 5: Key A = v3<br/>(← Newest value for key A)"]
        B6["Offset 6: Key B = null<br/>(← Tombstone delete key B)"]
        
        B0 --> B1 --> B2 --> B3 --> B4 --> B5 --> B6
    end
    
    subgraph After["After Compaction"]
        A3["Offset 3: Key C = v1<br/>(← Only value for key C)"]
        A5["Offset 5: Key A = v3<br/>(← Latest value for key A)"]
        A6["Offset 6: Key B = null<br/>(← Tombstone retained for delete.retention.ms)"]
        
        A3 --> A5 --> A6
    end
    
    Before -->|Compaction| After
    
    Uses["Use Cases:<br/>• Changelog topics (database CDC)<br/>• State stores (Kafka Streams)<br/>• Configuration topics"]
    After --> Uses
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│                          Log Compaction                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Before Compaction (cleanup.policy=compact):                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ Offset │ Key  │ Value                                               │     │
│  ├────────┼──────┼──────────────────────────────────────────────────────│     │
│  │   0    │  A   │ v1                                                  │     │
│  │   1    │  B   │ v1                                                  │     │
│  │   2    │  A   │ v2    ← Newer value for key A                      │     │
│  │   3    │  C   │ v1                                                  │     │
│  │   4    │  B   │ v2    ← Newer value for key B                      │     │
│  │   5    │  A   │ v3    ← Newest value for key A                     │     │
│  │   6    │  B   │ null  ← Tombstone (delete key B)                   │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  After Compaction:                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ Offset │ Key  │ Value                                               │     │
│  ├────────┼──────┼──────────────────────────────────────────────────────│     │
│  │   3    │  C   │ v1    ← Only value for key C                       │     │
│  │   5    │  A   │ v3    ← Latest value for key A                     │     │
│  │   6    │  B   │ null  ← Tombstone retained for delete.retention.ms │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  Use Cases:                                                                   │
│  ├── Changelog topics (database CDC)                                         │
│  ├── State stores (Kafka Streams)                                            │
│  └── Configuration topics                                                    │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

</details>
```

---

## 5. Controller Architecture

### KRaft Mode (No ZooKeeper)

```mermaid
flowchart TB
    subgraph ControllerQuorum["Controller Quorum (Raft Consensus)"]
        C1["Controller 1<br/>(Leader)"]
        C2["Controller 2<br/>(Follower)"]
        C3["Controller 3<br/>(Follower)"]
        
        RaftLog["Raft Log (Metadata)<br/>[Topic Created]<br/>[Partition Assigned]<br/>[Broker Registered]<br/>[Leader Elected]"]
        
        C1 --> C2
        C2 --> C3
        C1 --> RaftLog
    end
    
    subgraph Brokers["Brokers"]
        B1["Broker 1<br/>Metadata Cache"]
        B2["Broker 2<br/>Metadata Cache"]
        B3["Broker 3<br/>Metadata Cache"]
        BN["Broker N<br/>Metadata Cache"]
    end
    
    ControllerQuorum -->|Metadata Updates| Brokers
    
    MetadataRecords["Metadata Records:<br/>• TopicRecord: Topic creation/deletion<br/>• PartitionRecord: Partition configuration<br/>• PartitionChangeRecord: Leader/ISR changes<br/>• BrokerRegistrationRecord: Broker join/leave<br/>• ConfigRecord: Configuration changes"]
    ControllerQuorum --> MetadataRecords
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│                      Controller Architecture (KRaft)                          │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Controller Quorum (Raft Consensus)                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │                                                                      │     │
│  │  ┌────────────┐    ┌────────────┐    ┌────────────┐                │     │
│  │  │ Controller │    │ Controller │    │ Controller │                │     │
│  │  │  1 (Leader)│───>│ 2 (Follower)│───>│ 3 (Follower)│               │     │
│  │  └─────┬──────┘    └────────────┘    └────────────┘                │     │
│  │        │                                                            │     │
│  │        │ Raft Log (Metadata)                                       │     │
│  │        │ ┌────────────────────────────────────────┐                │     │
│  │        │ │ [Topic Created] [Partition Assigned]   │                │     │
│  │        │ │ [Broker Registered] [Leader Elected]   │                │     │
│  │        │ └────────────────────────────────────────┘                │     │
│  │        │                                                            │     │
│  └────────┼────────────────────────────────────────────────────────────┘     │
│           │                                                                   │
│           │ Metadata Updates                                                  │
│           ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │                        Brokers                                       │     │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐                    │     │
│  │  │Broker 1│  │Broker 2│  │Broker 3│  │Broker N│                    │     │
│  │  │        │  │        │  │        │  │        │                    │     │
│  │  │Metadata│  │Metadata│  │Metadata│  │Metadata│                    │     │
│  │  │ Cache  │  │ Cache  │  │ Cache  │  │ Cache  │                    │     │
│  │  └────────┘  └────────┘  └────────┘  └────────┘                    │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  Metadata Records:                                                            │
│  ├── TopicRecord: Topic creation/deletion                                    │
│  ├── PartitionRecord: Partition configuration                                │
│  ├── PartitionChangeRecord: Leader/ISR changes                              │
│  ├── BrokerRegistrationRecord: Broker join/leave                            │
│  └── ConfigRecord: Configuration changes                                     │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

</details>
```

### Leader Election

```java
// Leader Election Process
public class LeaderElection {
    
    /**
     * Triggered when:
     * - Broker fails (detected via heartbeat timeout)
     * - Broker shuts down (controlled shutdown)
     * - Partition reassignment
     */
    public void electLeader(String topic, int partition) {
        // Get current partition state
        PartitionState state = metadataCache.getPartition(topic, partition);
        List<Integer> isr = state.getIsr();
        List<Integer> replicas = state.getReplicas();
        
        // Preferred leader election (if enabled)
        int preferredLeader = replicas.get(0);
        if (isr.contains(preferredLeader) && isAlive(preferredLeader)) {
            setLeader(topic, partition, preferredLeader);
            return;
        }
        
        // Elect from ISR (maintains consistency)
        for (int replica : isr) {
            if (isAlive(replica)) {
                setLeader(topic, partition, replica);
                return;
            }
        }
        
        // Unclean leader election (if enabled, may lose data)
        if (uncleanLeaderElectionEnabled) {
            for (int replica : replicas) {
                if (isAlive(replica)) {
                    log.warn("Unclean leader election for {}-{}", topic, partition);
                    setLeader(topic, partition, replica);
                    return;
                }
            }
        }
        
        // No leader available
        setLeader(topic, partition, -1);
        log.error("No leader available for {}-{}", topic, partition);
    }
}
```

---

## 6. Mermaid Diagrams

### System Architecture

```mermaid
graph TB
    subgraph Producers
        P1[Producer 1]
        P2[Producer 2]
        P3[Producer N]
    end
    
    subgraph "Broker Cluster"
        B1[Broker 1]
        B2[Broker 2]
        B3[Broker 3]
        BN[Broker N]
    end
    
    subgraph "Controller Quorum"
        C1[Controller 1<br/>Leader]
        C2[Controller 2]
        C3[Controller 3]
    end
    
    subgraph Consumers
        CG1[Consumer Group A]
        CG2[Consumer Group B]
    end
    
    P1 --> B1
    P2 --> B2
    P3 --> B3
    
    B1 <--> B2
    B2 <--> B3
    B3 <--> BN
    
    C1 --> B1
    C1 --> B2
    C1 --> B3
    C1 --> BN
    
    C1 <--> C2
    C2 <--> C3
    
    B1 --> CG1
    B2 --> CG1
    B3 --> CG2
    BN --> CG2
```

### Message Flow Sequence

```mermaid
sequenceDiagram
    participant P as Producer
    participant L as Leader Broker
    participant F1 as Follower 1
    participant F2 as Follower 2
    participant C as Consumer
    
    P->>L: Produce(records, acks=all)
    L->>L: Write to log
    
    par Replication
        F1->>L: Fetch
        L->>F1: Records
        F1->>F1: Write to log
    and
        F2->>L: Fetch
        L->>F2: Records
        F2->>F2: Write to log
    end
    
    L->>L: Update High Watermark
    L->>P: ProduceResponse(offset)
    
    C->>L: Fetch(offset)
    L->>C: Records (up to HW)
    C->>L: OffsetCommit
```

### Partition Distribution

```mermaid
graph LR
    subgraph "Topic: orders (RF=3)"
        subgraph "Partition 0"
            P0L[Broker 1<br/>Leader]
            P0F1[Broker 2<br/>Follower]
            P0F2[Broker 3<br/>Follower]
        end
        
        subgraph "Partition 1"
            P1L[Broker 2<br/>Leader]
            P1F1[Broker 3<br/>Follower]
            P1F2[Broker 1<br/>Follower]
        end
        
        subgraph "Partition 2"
            P2L[Broker 3<br/>Leader]
            P2F1[Broker 1<br/>Follower]
            P2F2[Broker 2<br/>Follower]
        end
    end
    
    P0L -.-> P0F1
    P0L -.-> P0F2
    P1L -.-> P1F1
    P1L -.-> P1F2
    P2L -.-> P2F1
    P2L -.-> P2F2
```

