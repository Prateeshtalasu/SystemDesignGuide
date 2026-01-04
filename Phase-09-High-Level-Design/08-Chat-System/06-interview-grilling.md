# Chat System - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, and level-specific expectations for a chat system design.

---

## Trade-off Questions

### Q1: Why WebSocket instead of HTTP long polling or Server-Sent Events?

**Answer:**

| Protocol          | Pros                          | Cons                           |
| ----------------- | ----------------------------- | ------------------------------ |
| **HTTP Polling**  | Simple, works everywhere      | High latency, wasteful         |
| **Long Polling**  | Lower latency than polling    | Connection overhead, one-way   |
| **SSE**           | Built-in reconnection         | One-way (server→client only)   |
| **WebSocket**     | Full-duplex, low overhead     | Requires sticky sessions       |

**Why WebSocket:**
1. **Bidirectional**: Chat needs both send and receive
2. **Low latency**: Single TCP connection, no HTTP overhead
3. **Efficient**: No repeated headers, binary framing
4. **Real-time**: Push messages instantly

**Trade-off:**
- WebSocket requires sticky sessions (same server for connection lifetime)
- Mitigation: Session registry in Redis for routing

**Fallback Strategy:**
```java
// Try WebSocket first, fall back to long polling
if (supportsWebSocket()) {
    connectWebSocket();
} else {
    startLongPolling();
}
```

---

### Q2: How do you ensure messages are delivered in order?

**Answer:**

**The Problem:**
- Network delays can cause out-of-order delivery
- Multiple servers may process messages concurrently
- Retries can cause reordering

**Solution: Sequence Numbers**

```java
// Server side: Generate sequence per conversation
long sequence = redis.incr("seq:" + conversationId);
message.setSequenceNumber(sequence);

// Client side: Buffer and reorder
public void onMessageReceived(Message msg) {
    buffer.add(msg);
    
    // Deliver in order
    while (buffer.hasNext(lastDelivered + 1)) {
        Message next = buffer.remove(lastDelivered + 1);
        display(next);
        lastDelivered++;
    }
    
    // Request missing if gap persists
    if (buffer.hasGap() && gapAge > 5_000) {
        requestMissing(lastDelivered + 1, buffer.firstKey());
    }
}
```

**Why Not Timestamps?**
- Clock skew between servers
- Same millisecond possible
- Sequence numbers are monotonic and gap-free

---

### Q3: How do you handle the "read receipt" for group messages?

**Answer:**

**The Challenge:**
- Group with 256 members
- Each member reads at different times
- Showing 256 individual read receipts is noisy

**Solution: Aggregated Read Receipts**

```java
// Store read status per user
public void markAsRead(String messageId, String userId) {
    redis.sadd("read:" + messageId, userId);
    
    // Notify sender periodically, not per read
    int readCount = redis.scard("read:" + messageId);
    if (shouldNotifySender(readCount)) {
        notifySender(messageId, readCount);
    }
}

// UI shows: "Read by 45 of 50"
// Detail view shows who read
```

**Trade-offs:**

| Approach              | Pros                    | Cons                    |
| --------------------- | ----------------------- | ----------------------- |
| Individual receipts   | Precise                 | Noisy, expensive        |
| Aggregated count      | Clean UI                | Less detail             |
| Sample (first 3)      | Shows names             | Incomplete              |

**WhatsApp approach:**
- Show "Delivered" when all received
- Show "Read" when all read
- Tap for individual details

---

### Q4: How do you handle messages when the recipient is offline?

**Answer:**

**Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    OFFLINE MESSAGE HANDLING                      │
└─────────────────────────────────────────────────────────────────┘

Sender → Server → Store in DB → Check recipient status
                                        │
                    ┌───────────────────┴───────────────────┐
                    ▼                                       ▼
              ONLINE                                   OFFLINE
                    │                                       │
                    ▼                                       ▼
           Push via WebSocket                    1. Queue in Redis
                                                 2. Send push notification
                                                        │
                                                        ▼
                                                 Recipient comes online
                                                        │
                                                        ▼
                                                 Sync queued messages
```

**Implementation:**

```java
public void deliverMessage(Message message, String recipientId) {
    // Always store first
    messageRepository.save(message);
    
    if (presenceService.isOnline(recipientId)) {
        // Real-time delivery
        pushToUser(recipientId, message);
    } else {
        // Queue for later
        messageQueue.add(recipientId, message);
        
        // Push notification
        pushNotificationService.send(recipientId, 
            "New message from " + message.getSenderName());
    }
}

// On reconnect
public void onUserConnected(String userId) {
    List<Message> pending = messageQueue.getAll(userId);
    for (Message msg : pending) {
        pushToUser(userId, msg);
    }
    messageQueue.clear(userId);
}
```

---

### Q5: Why Cassandra for messages instead of PostgreSQL?

**Answer:**

| Aspect              | PostgreSQL                    | Cassandra                      |
| ------------------- | ----------------------------- | ------------------------------ |
| Write throughput    | ~10K writes/sec               | ~100K writes/sec               |
| Read pattern        | Flexible queries              | Partition key required         |
| Scaling             | Vertical + read replicas      | Horizontal (linear)            |
| Consistency         | Strong (ACID)                 | Tunable (eventual default)     |

**Why Cassandra for Messages:**
1. **High write volume**: Billions of messages/day
2. **Time-series pattern**: Messages ordered by time within conversation
3. **Partition key**: `conversation_id` naturally groups related data
4. **Horizontal scale**: Add nodes as messages grow

**Trade-off:**
- No joins (denormalize user info into messages)
- No ad-hoc queries (design tables for access patterns)
- Eventual consistency (acceptable for chat)

---

### Q6: How do you handle end-to-end encryption?

**Answer:**

**Signal Protocol Overview:**
1. **X3DH Key Exchange**: Establish shared secret without prior communication
2. **Double Ratchet**: Forward secrecy - new keys for each message
3. **AES-256-GCM**: Symmetric encryption for actual messages

**Key Challenges:**

| Challenge              | Solution                                      |
| ---------------------- | --------------------------------------------- |
| Key distribution       | Pre-keys uploaded to server                   |
| Multi-device           | Each device has own keys, encrypt per device  |
| Group chat             | Sender Keys protocol (encrypt once)           |
| Key verification       | Safety numbers, QR codes                      |
| Server search          | Not possible (feature trade-off)              |

**Trade-off:**
- Server can't read messages → can't do server-side spam filtering
- Mitigation: Client-side ML models for spam detection

---

## Level-Specific Expectations

### L4 (Entry-Level)

**What's Expected:**
- Understand WebSocket basics
- Simple message flow
- Basic database schema

**Sample Answer Quality:**

> "Users connect via WebSocket to a chat server. When Alice sends a message to Bob, the server stores it in a database and pushes it to Bob's connection. If Bob is offline, we store the message and deliver when he comes online."

**Red Flags:**
- No mention of message ordering
- Doesn't understand connection management
- No discussion of scale

---

### L5 (Mid-Level)

**What's Expected:**
- WebSocket connection management
- Message ordering with sequence numbers
- Delivery guarantees
- Presence system design

**Sample Answer Quality:**

> "I'd use WebSocket for real-time bidirectional communication. Each server handles ~100K connections, so we need about 1,500 servers for 150M concurrent users.

> For message ordering, I'd use sequence numbers per conversation, stored in Redis. The client buffers messages and delivers them in order, requesting missing messages if there's a gap.

> For delivery guarantees, I'd use at-least-once delivery with client-side deduplication. Messages are stored in Cassandra before acknowledging to the sender. If the recipient is offline, messages are queued in Redis and delivered on reconnection, plus a push notification.

> For presence, I'd use Redis with TTL. Each connection refreshes the TTL with heartbeats. When all devices disconnect, the user is marked offline and we broadcast to relevant contacts."

**Red Flags:**
- Can't explain message ordering
- No understanding of delivery guarantees
- Missing presence design

---

### L6 (Senior)

**What's Expected:**
- System evolution and trade-offs
- End-to-end encryption
- Group chat optimization
- Cross-region architecture

**Sample Answer Quality:**

> "Let me walk through the evolution. For MVP, we'd use a simple WebSocket server with PostgreSQL for messages. This works for 100K users but doesn't scale.

> For scale, we need to separate concerns: stateless chat servers for connections, Redis for session routing and presence, Cassandra for message storage (optimized for time-series writes), and Kafka for reliable async processing.

> For message ordering, sequence numbers per conversation are essential. But there's a trade-off: centralized sequence generation (Redis INCR) is simple but becomes a bottleneck. For high-volume conversations, we can use range allocation - allocate 100 sequence numbers at a time to reduce Redis calls.

> For group chats, the fan-out strategy matters. Small groups (< 100 members): fan-out on write - push to all members immediately. Large groups: fan-out on read - store once, members pull when they open the group. This prevents a single message from generating thousands of writes.

> For end-to-end encryption, we'd use the Signal Protocol. Key exchange happens on first message, then messages are encrypted with rotating keys. The server never sees plaintext - this is critical for privacy but means we can't do server-side search.

> For global scale, we'd deploy regionally. Messages are written to the local region first, then async replicated. Cross-region chats have higher latency (~200ms vs ~50ms) but this is acceptable for chat."

**Red Flags:**
- Can't discuss trade-offs
- No understanding of E2E encryption
- Missing group chat optimization

---

## Common Interviewer Pushbacks

### "What if a user has 10,000 contacts?"

**Response:**
"Presence broadcasting to 10,000 contacts would be expensive. We optimize by only broadcasting to contacts who have an active chat window with this user. Others will fetch presence on-demand when they open the chat. We also batch presence updates - instead of 10,000 individual messages, we publish to a Redis channel and let each server deliver to its local connections."

### "How do you prevent message spam?"

**Response:**
"We implement rate limiting at multiple levels:
1. **Per-user rate limit**: Max 100 messages per minute
2. **Per-conversation rate limit**: Max 50 messages per minute per conversation
3. **Content filtering**: Block known spam patterns
4. **Reputation system**: New accounts have stricter limits

For implementation, we use a sliding window rate limiter in Redis:
```java
boolean allowed = rateLimiter.tryAcquire(userId, 100, Duration.ofMinutes(1));
```

If rate limit exceeded, we return an error and the client should back off."

### "What about message editing and deletion?"

**Response:**
"For editing, we store the edit as a new event with a reference to the original message. Clients receive the edit event and update their local copy. We keep edit history for compliance.

For deletion, we have two options:
1. **Soft delete**: Mark as deleted, hide in UI, but keep for compliance
2. **Hard delete**: Actually remove (for 'delete for everyone')

With E2E encryption, 'delete for everyone' is tricky - we can send a delete signal, but can't force the recipient's device to delete. We rely on client cooperation."

### "How do you handle media messages (images, videos)?"

**Response:**
"Media follows a different flow:
1. Client uploads media to CDN/S3 (direct upload with pre-signed URL)
2. Client sends message with media_id reference
3. Recipients download media from CDN on demand
4. Thumbnails are pre-generated for quick preview

For E2E encrypted media:
1. Client encrypts media locally
2. Uploads encrypted blob
3. Sends encryption key in message (which is E2E encrypted)
4. Recipient decrypts media locally"

### "What if WebSocket isn't supported?"

**Response:**
"We implement graceful fallback:
1. **Primary**: WebSocket
2. **Fallback 1**: Server-Sent Events + HTTP POST for sending
3. **Fallback 2**: Long polling

Detection happens on connection:
```java
if (supportsWebSocket()) {
    useWebSocket();
} else if (supportsSSE()) {
    useSSEWithHTTPPost();
} else {
    useLongPolling();
}
```

Long polling has higher latency (~1-2 seconds) but works everywhere."

---

## Summary

| Question Type       | Key Points to Cover                                |
| ------------------- | -------------------------------------------------- |
| Trade-offs          | WebSocket vs alternatives, ordering, delivery      |
| Scaling             | Connection management, regional deployment         |
| Failures            | Server crash, Redis failure, delivery timeout      |
| Security            | E2E encryption, rate limiting                      |
| Optimization        | Group fan-out, presence broadcasting               |
