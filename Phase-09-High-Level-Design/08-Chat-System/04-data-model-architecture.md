# Chat System - Data Model & Architecture

## Database Choices

| Data Type           | Database       | Rationale                                |
| ------------------- | -------------- | ---------------------------------------- |
| Messages            | Cassandra      | High write volume, time-series           |
| User profiles       | PostgreSQL     | ACID, complex queries                    |
| Conversations       | PostgreSQL     | Relational data                          |
| Message queue       | Kafka          | Reliable async delivery                  |
| Sessions/Presence   | Redis          | Fast reads, TTL support                  |
| Media metadata      | PostgreSQL     | Relational to messages                   |
| Media files         | S3/Object Store| Blob storage                             |

---

## Consistency Model

**CAP Theorem Tradeoff:**

We choose **Availability + Partition Tolerance (AP)** for messages, **Consistency + Partition Tolerance (CP)** for user data:

- **Messages (Cassandra)**: AP - High availability for message delivery, eventual consistency acceptable
- **User Profiles (PostgreSQL)**: CP - Strong consistency required for user data integrity
- **Presence (Redis)**: AP - Real-time presence can tolerate brief inconsistencies

**ACID vs BASE:**

**ACID (Strong Consistency) for:**
- User profile updates (PostgreSQL)
- Conversation metadata (PostgreSQL)
- Message delivery status (when using PostgreSQL for critical status)

**BASE (Eventual Consistency) for:**
- Message replication across regions (Cassandra)
- Presence status across multiple devices (Redis)
- Read receipts synchronization (eventual)

**Per-Operation Consistency Guarantees:**

| Operation | Consistency Level | Rationale |
|-----------|------------------|-----------|
| Send message | Eventual (Cassandra) | High write throughput, eventual delivery acceptable |
| Read conversation | Eventual (Cassandra) | May see messages out of order briefly during replication |
| User profile update | Strong (PostgreSQL) | Must be immediately consistent across all reads |
| Presence status | Eventual (Redis) | Real-time updates, brief inconsistencies acceptable |
| Message delivery status | Strong (PostgreSQL) | Critical for delivery guarantees |

### Database Transactions

**Transaction Isolation Level:**

PostgreSQL transactions use **READ_COMMITTED** isolation level (default) for most operations, with **REPEATABLE_READ** for critical message sequence operations.

**Isolation Level Configuration:**

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public Conversation getConversation(String conversationId) {
    // READ_COMMITTED allows reading committed data
    // Prevents dirty reads but allows non-repeatable reads
    return conversationRepository.findById(conversationId);
}

@Transactional(isolation = Isolation.REPEATABLE_READ)
public Message sendMessage(SendMessageRequest request) {
    // REPEATABLE_READ ensures sequence numbers are allocated correctly
    // Prevents non-repeatable reads during message ordering
    long sequenceNumber = sequenceService.getNextSequence(request.getConversationId());
    // ... create message with sequence number
}
```

**Why READ_COMMITTED for Most Operations:**

- **Performance**: READ_COMMITTED has lower locking overhead than REPEATABLE_READ or SERIALIZABLE
- **Sufficient for reads**: Prevents dirty reads, which is sufficient for reading conversation lists and user profiles
- **Better concurrency**: Allows more concurrent transactions

**Why REPEATABLE_READ for Message Ordering:**

- **Sequence number allocation**: When allocating sequence numbers, we need to ensure no other transaction reads a different sequence number value
- **Message ordering guarantee**: REPEATABLE_READ ensures that if a transaction reads the current max sequence number, it won't see a different value if it reads again (non-repeatable read prevention)
- **Ordering is guaranteed**: Sequence numbers are allocated within transactions using REPEATABLE_READ, ensuring consistent ordering even under concurrent message sends

**How Ordering is Guaranteed:**

1. Sequence numbers are allocated using Redis INCR (atomic operation) or within a REPEATABLE_READ transaction
2. Messages are stored with sequence_number in Cassandra
3. Queries order by sequence_number within each conversation partition
4. The composite index `idx_messages_conv_seq` on `(conversation_id, sequence_number)` ensures efficient ordered retrieval

---

## Messages Table (Cassandra)

```sql
-- Partition by conversation_id for efficient conversation queries
CREATE TABLE messages (
    conversation_id TEXT,
    message_id TIMEUUID,
    sender_id TEXT,
    content_type TEXT,
    content TEXT,
    media_id TEXT,
    reply_to_id TEXT,
    sequence_number BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- Index for ordering messages by sequence_number within a conversation
-- This is needed for efficient queries that retrieve messages in sequence order
CREATE INDEX idx_messages_conv_seq ON messages(conversation_id, sequence_number);

-- For user's recent messages across conversations
CREATE TABLE user_messages (
    user_id TEXT,
    conversation_id TEXT,
    message_id TIMEUUID,
    is_sender BOOLEAN,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, created_at, message_id)
) WITH CLUSTERING ORDER BY (created_at DESC, message_id DESC);
```

---

## Users Table (PostgreSQL)

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    user_id VARCHAR(50) UNIQUE NOT NULL,
    
    -- Profile
    username VARCHAR(50) UNIQUE NOT NULL,
    display_name VARCHAR(100),
    avatar_url TEXT,
    phone_number VARCHAR(20) UNIQUE,
    
    -- Settings
    settings JSONB DEFAULT '{}',
    
    -- Encryption
    public_key TEXT,  -- For E2E encryption
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_seen_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_users_phone ON users(phone_number);
CREATE INDEX idx_users_username ON users(username);
```

---

## Conversations Table (PostgreSQL)

```sql
CREATE TABLE conversations (
    id BIGSERIAL PRIMARY KEY,
    conversation_id VARCHAR(50) UNIQUE NOT NULL,
    
    -- Type
    type VARCHAR(20) NOT NULL,  -- 'direct', 'group'
    
    -- Group info (null for direct)
    name VARCHAR(100),
    avatar_url TEXT,
    
    -- Metadata
    created_by BIGINT REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Last message (denormalized for list view)
    last_message_id VARCHAR(50),
    last_message_at TIMESTAMP WITH TIME ZONE,
    last_message_preview TEXT
);

CREATE INDEX idx_conversations_updated ON conversations(updated_at DESC);
```

---

## Conversation Participants (PostgreSQL)

```sql
CREATE TABLE conversation_participants (
    id BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT REFERENCES conversations(id),
    user_id BIGINT REFERENCES users(id),
    
    -- Role
    role VARCHAR(20) DEFAULT 'member',  -- 'admin', 'member'
    
    -- Read state
    last_read_message_id VARCHAR(50),
    last_read_at TIMESTAMP WITH TIME ZONE,
    
    -- Notification settings
    muted_until TIMESTAMP WITH TIME ZONE,
    
    -- Timestamps
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(conversation_id, user_id)
);

CREATE INDEX idx_participants_user ON conversation_participants(user_id);
CREATE INDEX idx_participants_conv ON conversation_participants(conversation_id);
```

---

## Message Delivery Status (Cassandra)

```sql
-- Track delivery status per recipient
CREATE TABLE message_status (
    message_id TEXT,
    user_id TEXT,
    status TEXT,  -- 'sent', 'delivered', 'read'
    updated_at TIMESTAMP,
    PRIMARY KEY (message_id, user_id)
);

-- For querying undelivered messages per user
CREATE TABLE pending_messages (
    user_id TEXT,
    message_id TEXT,
    conversation_id TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, created_at, message_id)
) WITH CLUSTERING ORDER BY (created_at ASC, message_id ASC);
```

---

## Session/Presence (Redis)

```
# User session (per device)
HSET session:{user_id}:{device_id}
    server_id "chat-server-42"
    connected_at "2024-01-20T15:30:00Z"
    last_active "2024-01-20T16:45:00Z"

# Set TTL for session
EXPIRE session:{user_id}:{device_id} 300  # 5 minutes

# User presence
SET presence:{user_id} "online"
EXPIRE presence:{user_id} 60  # 1 minute, refreshed by heartbeat

# User's active devices
SADD user_devices:{user_id} "device_123" "device_456"

# Conversation typing indicators
SETEX typing:{conversation_id}:{user_id} 5 "1"  # 5 second TTL
```

---

## Undelivered Message Queue (Redis)

```
# Queue of messages pending delivery per user
LPUSH pending:{user_id} "{message_json}"

# Get pending messages
LRANGE pending:{user_id} 0 -1

# Remove after delivery
LREM pending:{user_id} 1 "{message_json}"
```

---

## Entity Relationship Diagram

```
┌─────────────────────┐       ┌─────────────────────┐
│       users         │       │   conversations     │
├─────────────────────┤       ├─────────────────────┤
│ id (PK)             │       │ id (PK)             │
│ user_id (unique)    │       │ conversation_id     │
│ username            │       │ type                │
│ phone_number        │       │ name                │
│ public_key          │       │ last_message_at     │
└─────────────────────┘       └─────────────────────┘
         │                             │
         │                             │
         └──────────┬──────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────┐
│            conversation_participants                 │
├─────────────────────────────────────────────────────┤
│ id (PK)                                             │
│ conversation_id (FK)                                │
│ user_id (FK)                                        │
│ role                                                │
│ last_read_message_id                                │
│ muted_until                                         │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  messages (Cassandra)                │
├─────────────────────────────────────────────────────┤
│ conversation_id (PK)                                │
│ message_id (CK)                                     │
│ sender_id                                           │
│ content_type                                        │
│ content                                             │
│ sequence_number                                     │
│ created_at                                          │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  REDIS CACHE                         │
├─────────────────────────────────────────────────────┤
│ session:{user_id}:{device_id} → connection info     │
│ presence:{user_id} → online/offline                 │
│ pending:{user_id} → undelivered messages            │
│ typing:{conv_id}:{user_id} → typing indicator       │
└─────────────────────────────────────────────────────┘
```

---

## Component Overview

| Component              | Purpose                                | Why It Exists                                    |
| ---------------------- | -------------------------------------- | ------------------------------------------------ |
| **Chat Server**        | Handles WebSocket connections          | Real-time bidirectional communication            |
| **Message Service**    | Message CRUD operations                | Business logic for messages                      |
| **Presence Service**   | Online/offline status                  | Real-time presence tracking                      |
| **Notification Service**| Push notifications                    | Reach offline users                              |
| **Sync Service**       | Multi-device synchronization           | Keep devices in sync                             |
| **Message Queue**      | Async message processing               | Decouple and scale                               |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    CLIENTS                                           │
│                    (Mobile Apps, Web Browsers, Desktop Apps)                        │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
                    ▼                   ▼                   ▼
            ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
            │    CDN      │     │ WebSocket   │     │ API Gateway │
            │   (Media)   │     │ Gateway     │     │   (REST)    │
            └─────────────┘     └──────┬──────┘     └──────┬──────┘
                                       │                   │
                                       │                   │
┌──────────────────────────────────────┼───────────────────┼──────────────────────────┐
│                                      ▼                   ▼                          │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                           CHAT SERVERS (Stateful)                            │   │
│  │                                                                              │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │   │
│  │  │ Chat Server 1│  │ Chat Server 2│  │ Chat Server 3│  │ Chat Server N│    │   │
│  │  │              │  │              │  │              │  │              │    │   │
│  │  │ Users: A,B,C │  │ Users: D,E,F │  │ Users: G,H,I │  │ Users: ...   │    │   │
│  │  │ Connections: │  │ Connections: │  │ Connections: │  │ Connections: │    │   │
│  │  │ 100K         │  │ 100K         │  │ 100K         │  │ 100K         │    │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘    │   │
│  │                                                                              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                              │
│  APPLICATION LAYER                   │                                              │
└──────────────────────────────────────┼──────────────────────────────────────────────┘
                                       │
            ┌──────────────────────────┼──────────────────────────┐
            ▼                          ▼                          ▼
┌───────────────────────┐  ┌───────────────────────┐  ┌───────────────────────┐
│   Message Service     │  │   Presence Service    │  │  Notification Service │
│                       │  │                       │  │                       │
│ - Store messages      │  │ - Track online users  │  │ - Push notifications  │
│ - Retrieve history    │  │ - Last seen           │  │ - APNs/FCM            │
│ - Delivery status     │  │ - Typing indicators   │  │ - SMS fallback        │
└───────────┬───────────┘  └───────────┬───────────┘  └───────────┬───────────┘
            │                          │                          │
            │                          │                          │
┌───────────┼──────────────────────────┼──────────────────────────┼───────────────────┐
│           ▼                          ▼                          ▼                   │
│  ┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐       │
│  │    Cassandra    │         │     Redis       │         │     Kafka       │       │
│  │   (Messages)    │         │ (Sessions/Cache)│         │  (Event Queue)  │       │
│  └─────────────────┘         └─────────────────┘         └─────────────────┘       │
│                                                                                     │
│  ┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐       │
│  │   PostgreSQL    │         │       S3        │         │    Zookeeper    │       │
│  │ (Users/Convs)   │         │    (Media)      │         │  (Coordination) │       │
│  └─────────────────┘         └─────────────────┘         └─────────────────┘       │
│                                                                                     │
│  DATA LAYER                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Message Flow (One-on-One)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          ONE-ON-ONE MESSAGE FLOW                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘

Alice (Sender)          Chat Server 1        Message Service       Chat Server 2       Bob (Recipient)
     │                       │                     │                     │                    │
     │ 1. Send message       │                     │                     │                    │
     │ ─────────────────────>│                     │                     │                    │
     │                       │                     │                     │                    │
     │                       │ 2. Store message    │                     │                    │
     │                       │ ───────────────────>│                     │                    │
     │                       │                     │                     │                    │
     │                       │                     │ 3. Write to         │                    │
     │                       │                     │    Cassandra        │                    │
     │                       │                     │ ──────────────>     │                    │
     │                       │                     │                     │                    │
     │                       │ 4. ACK stored       │                     │                    │
     │                       │ <───────────────────│                     │                    │
     │                       │                     │                     │                    │
     │ 5. ACK sent ✓         │                     │                     │                    │
     │ <─────────────────────│                     │                     │                    │
     │                       │                     │                     │                    │
     │                       │ 6. Lookup Bob's     │                     │                    │
     │                       │    connection       │                     │                    │
     │                       │ ───────────────────────────────────────>  │                    │
     │                       │                     │                     │                    │
     │                       │ 7. Bob on Server 2  │                     │                    │
     │                       │ <───────────────────────────────────────  │                    │
     │                       │                     │                     │                    │
     │                       │ 8. Route message to Server 2              │                    │
     │                       │ ─────────────────────────────────────────>│                    │
     │                       │                     │                     │                    │
     │                       │                     │                     │ 9. Push to Bob     │
     │                       │                     │                     │ ──────────────────>│
     │                       │                     │                     │                    │
     │                       │                     │                     │ 10. ACK received   │
     │                       │                     │                     │ <──────────────────│
     │                       │                     │                     │                    │
     │                       │ 11. Delivery status │                     │                    │
     │                       │ <─────────────────────────────────────────│                    │
     │                       │                     │                     │                    │
     │ 12. Delivered ✓✓      │                     │                     │                    │
     │ <─────────────────────│                     │                     │                    │
```

---

## Group Message Fan-out

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          GROUP MESSAGE FAN-OUT                                       │
└─────────────────────────────────────────────────────────────────────────────────────┘

Alice sends message to "Family" group (50 members)
                    │
                    ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│  CHAT SERVER (Alice's connection)                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  1. Receive message                                                          │  │
│  │  2. Validate sender is group member                                          │  │
│  │  3. Generate message_id and sequence_number                                  │  │
│  │  4. Store message (single write)                                             │  │
│  │  5. ACK to Alice                                                             │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│  FAN-OUT SERVICE                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  1. Get group members (50 users)                                             │  │
│  │  2. Lookup connection info for each member                                   │  │
│  │                                                                              │  │
│  │     Member Status:                                                           │  │
│  │     ├── 30 ONLINE (on various chat servers)                                 │  │
│  │     └── 20 OFFLINE                                                          │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
┌───────────────────┐   ┌───────────────────┐
│  ONLINE DELIVERY  │   │  OFFLINE HANDLING │
│                   │   │                   │
│  For each online: │   │  For each offline:│
│  1. Find server   │   │  1. Queue message │
│  2. Route message │   │  2. Send push     │
│  3. Get ACK       │   │     notification  │
│                   │   │                   │
│  ┌─────────────┐  │   │  ┌─────────────┐  │
│  │Chat Server 1│  │   │  │   Kafka     │  │
│  │ → 10 users  │  │   │  │  (Queue)    │  │
│  └─────────────┘  │   │  └─────────────┘  │
│  ┌─────────────┐  │   │         │        │
│  │Chat Server 2│  │   │         ▼        │
│  │ → 8 users   │  │   │  ┌─────────────┐  │
│  └─────────────┘  │   │  │Push Service │  │
│  ┌─────────────┐  │   │  │ APNs/FCM    │  │
│  │Chat Server 3│  │   │  └─────────────┘  │
│  │ → 12 users  │  │   │                   │
│  └─────────────┘  │   │                   │
└───────────────────┘   └───────────────────┘
```

---

## Presence System

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          PRESENCE SYSTEM                                             │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│  USER CONNECTS                                                                       │
│                                                                                      │
│  User Alice connects to Chat Server                                                  │
│                    │                                                                 │
│                    ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  1. WebSocket handshake                                                      │   │
│  │  2. Authenticate token                                                       │   │
│  │  3. Register connection in Redis:                                            │   │
│  │     HSET session:alice:device_1 server_id "chat-server-42"                  │   │
│  │     SADD user_devices:alice "device_1"                                      │   │
│  │  4. Set presence:                                                            │   │
│  │     SET presence:alice "online"                                             │   │
│  │     EXPIRE presence:alice 60                                                │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  PRESENCE BROADCAST                                                                  │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  1. Get Alice's contacts who are online                                      │   │
│  │  2. For each online contact:                                                 │   │
│  │     - Find their chat server                                                 │   │
│  │     - Send presence update: "Alice is online"                               │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Optimization: Don't broadcast to all contacts                                       │
│  - Only broadcast to users who have chat open with Alice                            │
│  - Others will fetch on-demand when opening chat                                    │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  HEARTBEAT MECHANISM                                                                 │
│                                                                                      │
│  Client                    Server                     Redis                          │
│    │                         │                          │                            │
│    │ PING (every 30s)        │                          │                            │
│    │ ───────────────────────>│                          │                            │
│    │                         │                          │                            │
│    │                         │ EXPIRE presence:alice 60 │                            │
│    │                         │ ─────────────────────────>                            │
│    │                         │                          │                            │
│    │ PONG                    │                          │                            │
│    │ <───────────────────────│                          │                            │
│    │                         │                          │                            │
│                                                                                      │
│  If no heartbeat for 60s → presence key expires → user considered offline           │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  DISCONNECT HANDLING                                                                 │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  On disconnect:                                                              │   │
│  │  1. Remove session: DEL session:alice:device_1                              │   │
│  │  2. Check if other devices: SCARD user_devices:alice                        │   │
│  │  3. If no other devices:                                                     │   │
│  │     - Set last_seen: SET last_seen:alice "2024-01-20T15:30:00Z"            │   │
│  │     - Set presence offline: SET presence:alice "offline"                    │   │
│  │     - Broadcast offline to relevant contacts                                 │   │
│  │  4. If other devices still connected:                                        │   │
│  │     - Keep presence as online                                                │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Multi-Device Sync

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          MULTI-DEVICE SYNC                                           │
└─────────────────────────────────────────────────────────────────────────────────────┘

Alice has 3 devices: Phone, Tablet, Laptop
                    │
                    ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│  SCENARIO: Alice sends message from Phone                                          │
│                                                                                    │
│  ┌─────────┐        ┌─────────┐        ┌─────────┐                               │
│  │  Phone  │        │ Tablet  │        │ Laptop  │                               │
│  │(Sender) │        │(Sync)   │        │(Sync)   │                               │
│  └────┬────┘        └────┬────┘        └────┬────┘                               │
│       │                  │                  │                                     │
│       │ Send "Hello"     │                  │                                     │
│       │ ─────────────────────────────────────────────────>                       │
│       │                  │                  │            │                        │
│       │                  │                  │      Chat Server                    │
│       │                  │                  │            │                        │
│       │                  │                  │  Store + Sync to other devices     │
│       │                  │                  │            │                        │
│       │                  │ Sync: "Hello"    │            │                        │
│       │                  │ <─────────────────────────────│                        │
│       │                  │                  │            │                        │
│       │                  │                  │ Sync: "Hello"                       │
│       │                  │                  │ <──────────│                        │
│       │                  │                  │                                     │
│  All devices now show the sent message                                            │
└───────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│  SCENARIO: Device comes online after being offline                                 │
│                                                                                    │
│  Laptop was offline for 2 hours, now reconnects                                   │
│                                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  1. Laptop connects to Chat Server                                           │  │
│  │  2. Laptop sends: "Last sync timestamp: 2024-01-20T13:30:00Z"               │  │
│  │  3. Server queries messages since that timestamp                             │  │
│  │  4. Server sends batch of missed messages                                    │  │
│  │  5. Laptop updates local state                                               │  │
│  │  6. Laptop sends ACKs for delivered messages                                 │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  Delta Sync Query:                                                                 │
│  SELECT * FROM messages                                                            │
│  WHERE conversation_id IN (user's conversations)                                   │
│  AND created_at > '2024-01-20T13:30:00Z'                                          │
│  ORDER BY created_at ASC                                                           │
│  LIMIT 1000                                                                        │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## End-to-End Encryption

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          END-TO-END ENCRYPTION (Signal Protocol)                     │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────────────┐
│  KEY EXCHANGE (First Message)                                                      │
│                                                                                    │
│  Alice                        Server                        Bob                    │
│    │                            │                            │                     │
│    │ 1. Get Bob's public keys   │                            │                     │
│    │ ──────────────────────────>│                            │                     │
│    │                            │                            │                     │
│    │ 2. Bob's identity key +    │                            │                     │
│    │    signed pre-key +        │                            │                     │
│    │    one-time pre-key        │                            │                     │
│    │ <──────────────────────────│                            │                     │
│    │                            │                            │                     │
│    │ 3. Generate shared secret  │                            │                     │
│    │    using X3DH protocol     │                            │                     │
│    │                            │                            │                     │
│    │ 4. Encrypt message with    │                            │                     │
│    │    derived key             │                            │                     │
│    │                            │                            │                     │
│    │ 5. Send encrypted message  │                            │                     │
│    │ ──────────────────────────>│                            │                     │
│    │                            │                            │                     │
│    │                            │ 6. Forward (can't read)    │                     │
│    │                            │ ────────────────────────────>                    │
│    │                            │                            │                     │
│    │                            │                            │ 7. Derive same     │
│    │                            │                            │    shared secret   │
│    │                            │                            │                     │
│    │                            │                            │ 8. Decrypt message │
└───────────────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────────────┐
│  MESSAGE ENCRYPTION                                                                │
│                                                                                    │
│  Plaintext: "Hello Bob!"                                                          │
│       │                                                                            │
│       ▼                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  1. Generate message key (from Double Ratchet)                               │  │
│  │  2. Encrypt with AES-256-GCM                                                 │  │
│  │  3. Add authentication tag                                                   │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│       │                                                                            │
│       ▼                                                                            │
│  Ciphertext: "a7f3b2c1..." (+ auth tag + key ID)                                  │
│                                                                                    │
│  Server sees: Encrypted blob, sender, recipient, timestamp                        │
│  Server CANNOT see: Message content                                               │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## Message Delivery Protocol

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          MESSAGE DELIVERY PROTOCOL                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘

Sender                    Server                    Recipient
  │                         │                          │
  │ 1. Send message         │                          │
  │ ───────────────────────>│                          │
  │                         │                          │
  │                         │ 2. Store message         │
  │                         │ ───────────────>         │
  │                         │                          │
  │ 3. ACK (sent)           │                          │
  │ <───────────────────────│                          │
  │                         │                          │
  │                         │ 4. Push to recipient     │
  │                         │ ─────────────────────────>
  │                         │                          │
  │                         │ 5. ACK (received)        │
  │                         │ <─────────────────────────
  │                         │                          │
  │ 6. Status: delivered    │                          │
  │ <───────────────────────│                          │
  │                         │                          │
  │                         │ 7. User reads message    │
  │                         │                          │
  │                         │ 8. Read receipt          │
  │                         │ <─────────────────────────
  │                         │                          │
  │ 9. Status: read         │                          │
  │ <───────────────────────│                          │
```

---

## Summary

| Component           | Technology/Approach                     |
| ------------------- | --------------------------------------- |
| Real-time protocol  | WebSocket with binary framing           |
| Message storage     | Cassandra (partitioned by conversation) |
| User/Conversation   | PostgreSQL                              |
| Session/Presence    | Redis with TTL                          |
| Message queue       | Kafka for async processing              |
| Delivery guarantee  | At-least-once with client dedup         |
| Ordering            | Sequence numbers per conversation       |

| Aspect              | Decision                                              |
| ------------------- | ----------------------------------------------------- |
| Connection protocol | WebSocket with heartbeat                              |
| Message routing     | Redis for session lookup, direct server-to-server     |
| Group fan-out       | Parallel delivery to online, queue for offline        |
| Presence            | Redis with TTL, heartbeat refresh                     |
| Multi-device        | Sync to all devices, delta sync on reconnect          |
| Encryption          | Signal Protocol (E2E)                                 |
| Failure handling    | Stateless servers, message queue for reliability      |

