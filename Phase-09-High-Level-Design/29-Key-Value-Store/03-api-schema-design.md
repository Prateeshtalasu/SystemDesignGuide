# Key-Value Store - API & Schema Design

## API Design Philosophy

The Key-Value Store provides a simple, fast API for storing and retrieving data. It supports multiple data structures and operations.

---

## Protocol: Redis Protocol (RESP)

**Why RESP:**
- Simple, text-based protocol
- Widely supported
- Efficient parsing
- Human-readable

**Example:**
```
SET user:123 "John Doe"
→ +OK

GET user:123
→ $8
→ John Doe
```

---

## Basic Operations

### Strings

**SET key value [EX seconds] [NX|XX]**
```
SET user:123 "John Doe"
SET user:123 "John Doe" EX 3600  # Expire in 1 hour
SET user:123 "John Doe" NX  # Only if not exists
```

**GET key**
```
GET user:123
→ "John Doe"
```

**DELETE key [key ...]**
```
DELETE user:123
→ (integer) 1
```

**EXISTS key [key ...]**
```
EXISTS user:123
→ (integer) 1
```

**TTL key**
```
TTL user:123
→ (integer) 3600  # Seconds until expiration
```

**EXPIRE key seconds**
```
EXPIRE user:123 3600
→ (integer) 1
```

---

## Data Structures

### Hashes

**HSET key field value [field value ...]**
```
HSET user:123 name "John" age "30"
→ (integer) 2
```

**HGET key field**
```
HGET user:123 name
→ "John"
```

**HGETALL key**
```
HGETALL user:123
→ 1) "name"
   2) "John"
   3) "age"
   4) "30"
```

### Lists

**LPUSH key value [value ...]**
```
LPUSH queue:orders "order_123"
→ (integer) 1
```

**RPOP key**
```
RPOP queue:orders
→ "order_123"
```

**LRANGE key start stop**
```
LRANGE queue:orders 0 -1
→ 1) "order_123"
   2) "order_456"
```

### Sets

**SADD key member [member ...]**
```
SADD tags:post:123 "tech" "programming"
→ (integer) 2
```

**SMEMBERS key**
```
SMEMBERS tags:post:123
→ 1) "tech"
   2) "programming"
```

**SISMEMBER key member**
```
SISMEMBER tags:post:123 "tech"
→ (integer) 1
```

### Sorted Sets

**ZADD key score member [score member ...]**
```
ZADD leaderboard 1000 "user_123"
→ (integer) 1
```

**ZRANGE key start stop [WITHSCORES]**
```
ZRANGE leaderboard 0 9 WITHSCORES
→ 1) "user_123"
   2) "1000"
```

**ZRANK key member**
```
ZRANK leaderboard user_123
→ (integer) 0
```

---

## Advanced Operations

### Transactions

**MULTI**
```
MULTI
SET key1 "value1"
SET key2 "value2"
EXEC
→ 1) OK
   2) OK
```

**WATCH key [key ...]**
```
WATCH user:123
MULTI
SET user:123 "new_value"
EXEC
→ Executes only if user:123 unchanged
```

### Pub/Sub

**PUBLISH channel message**
```
PUBLISH notifications "New message"
→ (integer) 5  # Number of subscribers
```

**SUBSCRIBE channel [channel ...]**
```
SUBSCRIBE notifications
→ 1) "subscribe"
   2) "notifications"
   3) (integer) 1
```

---

## Error Responses

**Error Format:**
```
-ERR error message
```

**Common Errors:**
- `-ERR wrong number of arguments`
- `-ERR no such key`
- `-ERR value is not an integer`
- `-ERR operation against a key holding the wrong kind of value`

---

## Pipelining

**Batch Operations:**
```
Client sends multiple commands without waiting for responses:
SET key1 "value1"
SET key2 "value2"
GET key1
GET key2

Server processes and returns all responses.
```

**Benefits:**
- Reduces network round-trips
- Improves throughput
- Atomic batch operations

---

## Lua Scripting

**EVAL script numkeys key [key ...] arg [arg ...]**
```
EVAL "return redis.call('GET', KEYS[1])" 1 user:123
→ "John Doe"
```

**Benefits:**
- Atomic operations
- Server-side processing
- Reduced network overhead

---

## Connection Management

**AUTH password**
```
AUTH mypassword
→ +OK
```

**PING**
```
PING
→ +PONG
```

**SELECT index**
```
SELECT 0  # Select database 0
→ +OK
```

---

## Persistence Commands

**BGSAVE**
```
BGSAVE
→ +Background saving started
```

**SAVE**
```
SAVE
→ +OK  # Blocks until complete
```

---

## Summary

**Protocol:** RESP (Redis Protocol)
**Operations:** Strings, Hashes, Lists, Sets, Sorted Sets
**Advanced:** Transactions, Pub/Sub, Lua Scripting
**Persistence:** RDB, AOF commands

---

## Next Steps

After API design, we'll design:
1. Data structures implementation
2. Persistence mechanisms
3. Replication and clustering
4. Memory management

