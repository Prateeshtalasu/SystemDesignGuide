# Chat System (WhatsApp/Slack) - Problem & Requirements

## Problem Statement

Design a real-time chat system that supports one-on-one messaging, group chats, online presence, and message synchronization across multiple devices. The system should handle billions of messages daily with low latency and high reliability.

---

## Functional Requirements

### Core Features

| Feature                  | Description                                           | Priority |
| ------------------------ | ----------------------------------------------------- | -------- |
| **One-on-One Messaging** | Send/receive messages between two users               | P0       |
| **Group Chat**           | Multi-user conversations (up to 256 members)          | P0       |
| **Message Delivery**     | Guaranteed delivery with status (sent, delivered, read)| P0      |
| **Online Presence**      | Show user online/offline/last seen status             | P0       |
| **Message Sync**         | Sync messages across multiple devices                 | P0       |
| **Push Notifications**   | Notify users of new messages when offline             | P0       |

### Extended Features

| Feature                  | Description                                           | Priority |
| ------------------------ | ----------------------------------------------------- | -------- |
| **Read Receipts**        | Show when messages are read                           | P1       |
| **Typing Indicators**    | Show when user is typing                              | P1       |
| **Media Sharing**        | Send images, videos, documents                        | P1       |
| **Message Search**       | Search through message history                        | P1       |
| **End-to-End Encryption**| Encrypt messages so only sender/receiver can read     | P1       |
| **Message Reactions**    | React to messages with emojis                         | P2       |
| **Voice/Video Calls**    | Real-time audio/video communication                   | P2       |

---

## Non-Functional Requirements

| Requirement      | Target                | Rationale                                      |
| ---------------- | --------------------- | ---------------------------------------------- |
| **Latency**      | < 100ms (P99)         | Real-time feel is critical for chat            |
| **Throughput**   | 500K messages/second  | WhatsApp-scale messaging                       |
| **Availability** | 99.99%                | Chat is critical communication                 |
| **Durability**   | 99.999999%            | Messages must never be lost                    |
| **Consistency**  | Eventual (ordered)    | Messages must arrive in order                  |
| **Scalability**  | 2B users, 100B msg/day| Global scale                                   |

---

## User Stories

### User Perspective

1. **As a user**, I want to send messages instantly so I can have real-time conversations.

2. **As a user**, I want to see when my message is delivered and read so I know the recipient received it.

3. **As a user**, I want to see who is online so I know when to expect quick responses.

4. **As a user**, I want my messages synced across all my devices so I can continue conversations anywhere.

5. **As a user**, I want to create group chats so I can communicate with multiple people at once.

### Business Perspective

1. **As a product manager**, I want reliable message delivery to maintain user trust.

2. **As a security officer**, I want end-to-end encryption to protect user privacy.

---

## Constraints & Assumptions

### Constraints

1. **Real-time**: Messages must feel instant (< 100ms)
2. **Ordering**: Messages must be delivered in order
3. **Durability**: Zero message loss tolerance
4. **Mobile-first**: Optimize for mobile networks

### Assumptions

1. Users have unique identifiers
2. Users can be on multiple devices simultaneously
3. Average message size is ~1 KB
4. Users are online ~4 hours/day on average

---

## Out of Scope

1. Voice/Video calling infrastructure
2. Sticker/GIF marketplace
3. Payment integration
4. Stories/Status feature
5. Spam/abuse detection ML

---

## Key Challenges

### 1. Real-time Message Delivery

**Challenge:** Deliver messages with < 100ms latency globally

**Solution:**
- WebSocket connections for real-time push
- Edge servers close to users
- Connection pooling and keep-alive

### 2. Message Ordering

**Challenge:** Ensure messages arrive in correct order

**Solution:**
- Sequence numbers per conversation
- Server-side ordering before delivery
- Client-side reordering buffer

### 3. Offline Message Handling

**Challenge:** Deliver messages when recipient comes online

**Solution:**
- Store-and-forward architecture
- Message queue per user
- Sync on reconnection

### 4. Group Message Fan-out

**Challenge:** Send message to 256 group members efficiently

**Solution:**
- Fan-out on write for small groups
- Fan-out on read for large groups
- Parallel delivery to online members

### 5. Multi-device Sync

**Challenge:** Keep all user devices in sync

**Solution:**
- Device-specific message queues
- Last-sync timestamp per device
- Delta sync on reconnection

---

## Success Metrics

| Metric              | Definition                              | Target        |
| ------------------- | --------------------------------------- | ------------- |
| **Delivery Latency**| Time from send to delivery              | < 100ms (P99) |
| **Delivery Rate**   | Messages successfully delivered         | > 99.99%      |
| **Connection Uptime**| WebSocket connection stability         | > 99.9%       |
| **Sync Latency**    | Time to sync on reconnection            | < 2 seconds   |
| **Message Loss**    | Messages lost                           | 0%            |

---

## Example Scenarios

### Scenario 1: One-on-One Message

```
Alice sends "Hello" to Bob
→ Client encrypts message (E2E)
→ WebSocket sends to chat server
→ Server stores in message store
→ Server looks up Bob's connections
→ If Bob online: Push via WebSocket
→ If Bob offline: Queue + Push notification
→ Bob receives message
→ Bob's client sends delivery receipt
→ Alice sees "Delivered" ✓✓
```

### Scenario 2: Group Message

```
Alice sends message to "Family" group (50 members)
→ Server receives message
→ Server stores once in message store
→ Server fans out to 50 members:
   - 30 online: Push via WebSocket
   - 20 offline: Queue + Push notification
→ Each member receives message
→ Delivery receipts aggregated
```

### Scenario 3: Multi-device Sync

```
Alice has phone and laptop
→ Alice sends message from phone
→ Server stores message
→ Server pushes to Bob
→ Server also pushes to Alice's laptop (sync)
→ Both Alice's devices show sent message
```

---

## Message States

```
┌─────────────────────────────────────────────────────────────────┐
│                    MESSAGE STATE MACHINE                         │
└─────────────────────────────────────────────────────────────────┘

    ┌─────────┐     Server      ┌───────────┐    Recipient    ┌────────┐
    │ SENDING │ ──────────────> │   SENT    │ ──────────────> │DELIVERED│
    └─────────┘     received    └───────────┘     received    └────────┘
         │                           │                             │
         │                           │                             │
         │ Failure                   │ Server                      │ Recipient
         │                           │ stored                      │ opened
         ▼                           ▼                             ▼
    ┌─────────┐                 ┌───────────┐                 ┌────────┐
    │ FAILED  │                 │  STORED   │                 │  READ  │
    └─────────┘                 └───────────┘                 └────────┘

UI Indicators:
- SENDING: ⏳ (clock)
- SENT: ✓ (single check)
- DELIVERED: ✓✓ (double check, gray)
- READ: ✓✓ (double check, blue)
- FAILED: ❌ (retry option)
```

---

## Summary

| Aspect              | Decision                                              |
| ------------------- | ----------------------------------------------------- |
| Real-time protocol  | WebSocket with fallback to long polling               |
| Message storage     | Store-and-forward with per-user queues                |
| Delivery guarantee  | At-least-once with deduplication                      |
| Ordering            | Sequence numbers per conversation                     |
| Encryption          | End-to-end encryption (Signal Protocol)               |
| Scale               | 2B users, 500K messages/second                        |

