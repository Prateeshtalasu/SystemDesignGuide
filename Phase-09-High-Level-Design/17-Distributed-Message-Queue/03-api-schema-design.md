# Distributed Message Queue - API & Schema Design

## 1. API Design Principles

### Protocol Overview

```
Primary Protocol: Binary protocol over TCP (like Kafka)
Secondary: REST API for admin operations
Port: 9092 (default)

Design Principles:
├── Request-Response model
├── Multiplexed connections
├── Correlation IDs for matching
├── Versioned API for compatibility
└── Batch operations for efficiency
```

### Request/Response Format

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Message Format                                        │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Request Header:                                                              │
│  ┌────────────┬────────────┬────────────┬────────────┬────────────┐         │
│  │ Length (4) │ API Key(2) │ Version(2) │ Corr ID(4) │ Client(var)│         │
│  └────────────┴────────────┴────────────┴────────────┴────────────┘         │
│                                                                               │
│  Response Header:                                                             │
│  ┌────────────┬────────────┐                                                 │
│  │ Length (4) │ Corr ID(4) │                                                 │
│  └────────────┴────────────┘                                                 │
│                                                                               │
│  API Keys:                                                                    │
│  ├── 0: Produce                                                              │
│  ├── 1: Fetch                                                                │
│  ├── 2: ListOffsets                                                          │
│  ├── 3: Metadata                                                             │
│  ├── 8: OffsetCommit                                                         │
│  ├── 9: OffsetFetch                                                          │
│  ├── 10: FindCoordinator                                                     │
│  ├── 11: JoinGroup                                                           │
│  ├── 12: Heartbeat                                                           │
│  ├── 13: LeaveGroup                                                          │
│  ├── 14: SyncGroup                                                           │
│  └── ...                                                                     │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Producer API

### Produce Request

```java
// Produce Request Structure
public class ProduceRequest {
    short acks;                    // -1=all, 0=none, 1=leader
    int timeoutMs;                 // Request timeout
    List<TopicData> topics;
    
    public static class TopicData {
        String topicName;
        List<PartitionData> partitions;
    }
    
    public static class PartitionData {
        int partition;
        RecordBatch records;       // Batch of messages
    }
}

// Record Batch Format
public class RecordBatch {
    long baseOffset;
    int batchLength;
    int partitionLeaderEpoch;
    byte magic;                    // Format version
    int crc;                       // Checksum
    short attributes;              // Compression, timestamp type
    int lastOffsetDelta;
    long firstTimestamp;
    long maxTimestamp;
    long producerId;               // For idempotence
    short producerEpoch;
    int baseSequence;              // For ordering
    List<Record> records;
}

// Individual Record
public class Record {
    int length;
    byte attributes;
    long timestampDelta;
    int offsetDelta;
    byte[] key;
    byte[] value;
    List<Header> headers;
}
```

### Produce Response

```java
public class ProduceResponse {
    List<TopicResponse> responses;
    int throttleTimeMs;
    
    public static class TopicResponse {
        String topicName;
        List<PartitionResponse> partitions;
    }
    
    public static class PartitionResponse {
        int partition;
        short errorCode;           // 0=success
        long baseOffset;           // First offset assigned
        long logAppendTimeMs;
        long logStartOffset;
    }
}

// Error Codes
public enum ErrorCode {
    NONE(0),
    UNKNOWN_TOPIC_OR_PARTITION(3),
    LEADER_NOT_AVAILABLE(5),
    NOT_LEADER_FOR_PARTITION(6),
    REQUEST_TIMED_OUT(7),
    MESSAGE_TOO_LARGE(10),
    RECORD_LIST_TOO_LARGE(18),
    NOT_ENOUGH_REPLICAS(19),
    NOT_ENOUGH_REPLICAS_AFTER_APPEND(20),
    INVALID_REQUIRED_ACKS(21),
    OUT_OF_ORDER_SEQUENCE_NUMBER(45),
    DUPLICATE_SEQUENCE_NUMBER(46);
}
```

### Producer Client Usage

```java
// High-level Producer API
public interface Producer<K, V> {
    
    // Send message asynchronously
    Future<RecordMetadata> send(ProducerRecord<K, V> record);
    
    // Send with callback
    Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback);
    
    // Flush pending messages
    void flush();
    
    // Begin transaction (for exactly-once)
    void beginTransaction();
    
    // Commit transaction
    void commitTransaction();
    
    // Abort transaction
    void abortTransaction();
    
    // Close producer
    void close();
}

// Usage Example
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("key.serializer", "StringSerializer");
props.put("value.serializer", "JsonSerializer");
props.put("acks", "all");
props.put("retries", 3);
props.put("enable.idempotence", true);

Producer<String, Order> producer = new KafkaProducer<>(props);

ProducerRecord<String, Order> record = new ProducerRecord<>(
    "orders",                    // topic
    order.getUserId(),           // key (for partitioning)
    order                        // value
);

producer.send(record, (metadata, exception) -> {
    if (exception == null) {
        System.out.printf("Sent to partition %d, offset %d%n",
            metadata.partition(), metadata.offset());
    } else {
        exception.printStackTrace();
    }
});
```

---

## 3. Consumer API

### Fetch Request

```java
public class FetchRequest {
    int replicaId;                 // -1 for consumers
    int maxWaitMs;                 // Long poll timeout
    int minBytes;                  // Min data to return
    int maxBytes;                  // Max data to return
    byte isolationLevel;           // READ_UNCOMMITTED or READ_COMMITTED
    List<TopicPartition> topics;
    
    public static class TopicPartition {
        String topic;
        List<PartitionFetch> partitions;
    }
    
    public static class PartitionFetch {
        int partition;
        long fetchOffset;          // Start offset
        int partitionMaxBytes;     // Max bytes for this partition
    }
}
```

### Fetch Response

```java
public class FetchResponse {
    int throttleTimeMs;
    short errorCode;
    int sessionId;
    List<TopicResponse> responses;
    
    public static class TopicResponse {
        String topic;
        List<PartitionResponse> partitions;
    }
    
    public static class PartitionResponse {
        int partition;
        short errorCode;
        long highWatermark;        // Latest committed offset
        long lastStableOffset;     // For transactions
        long logStartOffset;
        List<AbortedTransaction> abortedTransactions;
        RecordBatch records;       // The actual messages
    }
}
```

### Consumer Client Usage

```java
// High-level Consumer API
public interface Consumer<K, V> {
    
    // Subscribe to topics
    void subscribe(Collection<String> topics);
    void subscribe(Pattern pattern);
    
    // Assign specific partitions
    void assign(Collection<TopicPartition> partitions);
    
    // Poll for records
    ConsumerRecords<K, V> poll(Duration timeout);
    
    // Commit offsets
    void commitSync();
    void commitAsync();
    void commitSync(Map<TopicPartition, OffsetAndMetadata> offsets);
    
    // Seek to offset
    void seek(TopicPartition partition, long offset);
    void seekToBeginning(Collection<TopicPartition> partitions);
    void seekToEnd(Collection<TopicPartition> partitions);
    
    // Get position
    long position(TopicPartition partition);
    
    // Pause/Resume
    void pause(Collection<TopicPartition> partitions);
    void resume(Collection<TopicPartition> partitions);
    
    // Close
    void close();
}

// Usage Example
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("group.id", "order-processor");
props.put("key.deserializer", "StringDeserializer");
props.put("value.deserializer", "JsonDeserializer");
props.put("auto.offset.reset", "earliest");
props.put("enable.auto.commit", false);

Consumer<String, Order> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("orders"));

while (true) {
    ConsumerRecords<String, Order> records = consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, Order> record : records) {
        System.out.printf("Received: partition=%d, offset=%d, key=%s%n",
            record.partition(), record.offset(), record.key());
        
        // Process order
        processOrder(record.value());
    }
    
    // Commit after processing
    consumer.commitSync();
}
```

---

## Base URL Structure

```
Binary Protocol: tcp://broker.example.com:9092
REST Admin API:  https://admin.queue.example.com/v1
```

---

## API Versioning Strategy

We use URL path versioning (`/v1/`, `/v2/`) for REST API and protocol versioning for binary protocol because:
- Easy to understand and implement
- Clear in logs and documentation
- Allows running multiple versions simultaneously

**Backward Compatibility Rules:**

Non-breaking changes (no version bump):
- Adding new optional fields
- Adding new endpoints
- Adding new error codes

Breaking changes (require new version):
- Removing fields
- Changing field types
- Changing endpoint paths
- Changing protocol format

**Deprecation Policy:**
1. Announce deprecation 6 months in advance
2. Return Deprecation header
3. Maintain old version for 12 months after new version release

---

## Rate Limiting Headers

Every REST API response includes rate limit information:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
```

**Rate Limits:**

| Endpoint Type | Requests/minute |
|---------------|-----------------|
| Admin Operations | 100 |
| Monitoring | 1000 |

---

## Error Model

All REST API error responses follow this standard envelope structure:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "field": "field_name",  // Optional
      "reason": "Specific reason"  // Optional
    },
    "request_id": "req_123456"  // For tracing
  }
}
```

**Error Codes Reference (REST API):**

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | INVALID_INPUT | Request validation failed |
| 400 | INVALID_TOPIC_NAME | Topic name is invalid |
| 400 | INVALID_PARTITION_COUNT | Partition count is invalid |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Topic or resource not found |
| 409 | CONFLICT | Topic already exists |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

**Binary Protocol Error Codes:**

| Code | Name | Description |
|------|------|-------------|
| 0 | NONE | Success |
| 1 | OFFSET_OUT_OF_RANGE | Requested offset invalid |
| 3 | UNKNOWN_TOPIC_OR_PARTITION | Topic doesn't exist |
| 5 | LEADER_NOT_AVAILABLE | No leader for partition |
| 6 | NOT_LEADER_FOR_PARTITION | Broker is not leader |
| 7 | REQUEST_TIMED_OUT | Request timed out |
| 10 | MESSAGE_TOO_LARGE | Message exceeds max size |
| 19 | NOT_ENOUGH_REPLICAS | ISR too small |
| 22 | ILLEGAL_GENERATION | Consumer group rebalancing |
| 25 | UNKNOWN_MEMBER_ID | Consumer needs to rejoin |
| 27 | REBALANCE_IN_PROGRESS | Consumer group rebalancing |

**Error Response Examples (REST API):**

```json
// 400 Bad Request - Invalid topic name
{
  "error": {
    "code": "INVALID_TOPIC_NAME",
    "message": "Topic name is invalid",
    "details": {
      "field": "name",
      "reason": "Topic name must match pattern: [a-zA-Z0-9._-]+"
    },
    "request_id": "req_abc123"
  }
}

// 409 Conflict - Topic exists
{
  "error": {
    "code": "CONFLICT",
    "message": "Topic already exists",
    "details": {
      "field": "name",
      "reason": "Topic 'orders' already exists"
    },
    "request_id": "req_xyz789"
  }
}

// 429 Rate Limited
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded. Please try again later.",
    "details": {
      "limit": 100,
      "remaining": 0,
      "reset_at": "2024-01-15T11:00:00Z"
    },
    "request_id": "req_def456"
  }
}
```

---

## Idempotency Implementation

**Idempotency-Key Header (REST Admin API):**

Clients must include `Idempotency-Key` header for all write operations in REST Admin API:
```http
Idempotency-Key: <uuid-v4>
```

**Deduplication Storage:**

Idempotency keys are stored in Redis with 24-hour TTL:
```
Key: idempotency:{key}
Value: Serialized response
TTL: 24 hours
```

**Retry Semantics:**

1. Client sends request with Idempotency-Key
2. Server checks Redis for existing key
3. If found: Return cached response (same status code + body)
4. If not found: Process request, cache response, return result
5. Retries with same key within 24 hours return cached response

**Binary Protocol Idempotency:**

For binary protocol (Producer API), idempotency is handled via:
- Producer ID and Producer Epoch (for exactly-once semantics)
- Sequence numbers per partition
- Deduplication on broker side using (producer_id, partition, sequence)

**Per-Endpoint Idempotency (REST Admin API):**

| Endpoint | Idempotent? | Mechanism |
|----------|-------------|-----------|
| POST /admin/topics | Yes | Idempotency-Key header |
| PUT /admin/topics/{name} | Yes | Idempotency-Key or version-based |
| DELETE /admin/topics/{name} | Yes | Safe to retry (idempotent by design) |

**Implementation Example (REST Admin API):**

```java
@PostMapping("/admin/topics")
public ResponseEntity<TopicResponse> createTopic(
        @RequestBody CreateTopicRequest request,
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {
    
    // Check for existing idempotency key
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        String cachedResponse = redisTemplate.opsForValue().get(cacheKey);
        if (cachedResponse != null) {
            TopicResponse response = objectMapper.readValue(cachedResponse, TopicResponse.class);
            return ResponseEntity.status(response.getStatus()).body(response);
        }
    }
    
    // Check if topic already exists
    if (topicService.topicExists(request.getName())) {
        throw new ConflictException("Topic already exists");
    }
    
    // Create topic
    Topic topic = topicService.createTopic(request, idempotencyKey);
    TopicResponse response = TopicResponse.from(topic);
    
    // Cache response if idempotency key provided
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        redisTemplate.opsForValue().set(
            cacheKey, 
            objectMapper.writeValueAsString(response),
            Duration.ofHours(24)
        );
    }
    
    return ResponseEntity.status(201).body(response);
}
```

---

## 4. Admin API

### Topic Management

```java
// Create Topic
public class CreateTopicsRequest {
    List<TopicConfig> topics;
    int timeoutMs;
    boolean validateOnly;
    
    public static class TopicConfig {
        String name;
        int numPartitions;
        short replicationFactor;
        Map<Integer, List<Integer>> replicaAssignment;  // Manual assignment
        Map<String, String> configs;  // Topic configs
    }
}

// Delete Topic
public class DeleteTopicsRequest {
    List<String> topicNames;
    int timeoutMs;
}

// Describe Topic
public class DescribeTopicsResponse {
    List<TopicDescription> topics;
    
    public static class TopicDescription {
        String name;
        boolean isInternal;
        List<PartitionInfo> partitions;
        
        public static class PartitionInfo {
            int partition;
            int leader;
            List<Integer> replicas;
            List<Integer> isr;  // In-sync replicas
        }
    }
}
```

### REST Admin API

```http
# Create Topic
POST /admin/topics
Content-Type: application/json

{
  "name": "orders",
  "partitions": 100,
  "replication_factor": 3,
  "configs": {
    "retention.ms": "604800000",
    "cleanup.policy": "delete",
    "compression.type": "lz4"
  }
}

Response: 201 Created
{
  "name": "orders",
  "partitions": 100,
  "replication_factor": 3,
  "created_at": "2024-01-15T10:30:00Z"
}

# List Topics
GET /admin/topics

Response: 200 OK
{
  "topics": [
    {
      "name": "orders",
      "partitions": 100,
      "replication_factor": 3
    },
    {
      "name": "payments",
      "partitions": 50,
      "replication_factor": 3
    }
  ]
}

# Describe Topic
GET /admin/topics/orders

Response: 200 OK
{
  "name": "orders",
  "partitions": [
    {
      "id": 0,
      "leader": 1,
      "replicas": [1, 2, 3],
      "isr": [1, 2, 3]
    },
    {
      "id": 1,
      "leader": 2,
      "replicas": [2, 3, 1],
      "isr": [2, 3, 1]
    }
  ],
  "configs": {
    "retention.ms": "604800000"
  }
}

# Delete Topic
DELETE /admin/topics/orders

Response: 204 No Content
```

---

## 5. Consumer Group Protocol

### Join Group

```java
public class JoinGroupRequest {
    String groupId;
    int sessionTimeoutMs;
    int rebalanceTimeoutMs;
    String memberId;              // Empty on first join
    String protocolType;          // "consumer"
    List<Protocol> protocols;     // Supported assignment strategies
}

public class JoinGroupResponse {
    short errorCode;
    int generationId;             // Group generation
    String protocolName;          // Selected protocol
    String leader;                // Group leader member ID
    String memberId;              // Assigned member ID
    List<Member> members;         // Only for leader
}
```

### Sync Group

```java
// Leader sends assignments, followers receive
public class SyncGroupRequest {
    String groupId;
    int generationId;
    String memberId;
    List<Assignment> assignments;  // Only from leader
}

public class SyncGroupResponse {
    short errorCode;
    byte[] assignment;            // Partition assignment for this member
}
```

### Heartbeat

```java
public class HeartbeatRequest {
    String groupId;
    int generationId;
    String memberId;
}

public class HeartbeatResponse {
    short errorCode;  // REBALANCE_IN_PROGRESS triggers rejoin
}
```

---

## 6. Data Storage Schema

### Log Segment Structure

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Log Segment Structure                                 │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Directory: /data/kafka-logs/{topic}-{partition}/                            │
│                                                                               │
│  Files per segment:                                                           │
│  ├── 00000000000000000000.log      # Message data                            │
│  ├── 00000000000000000000.index    # Offset index                            │
│  ├── 00000000000000000000.timeindex # Time index                             │
│  └── 00000000000000000000.txnindex # Transaction index (if applicable)       │
│                                                                               │
│  Log File Format:                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ RecordBatch │ RecordBatch │ RecordBatch │ ... │ RecordBatch │      │     │
│  │  (offset 0) │  (offset 10)│  (offset 25)│     │  (offset N) │      │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  Index File Format (sparse index):                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ Relative Offset (4) │ Physical Position (4) │                       │     │
│  ├─────────────────────┼──────────────────────┤                       │     │
│  │         0           │          0           │                       │     │
│  │       100           │       4096           │                       │     │
│  │       200           │       8192           │                       │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  Time Index Format:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ Timestamp (8) │ Relative Offset (4) │                               │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Metadata Schema

```sql
-- Topics (stored in controller)
CREATE TABLE topics (
    name VARCHAR(255) PRIMARY KEY,
    num_partitions INT NOT NULL,
    replication_factor SMALLINT NOT NULL,
    configs JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Partitions
CREATE TABLE partitions (
    topic_name VARCHAR(255) NOT NULL,
    partition_id INT NOT NULL,
    leader_broker_id INT,
    leader_epoch INT DEFAULT 0,
    replicas INT[] NOT NULL,
    isr INT[] NOT NULL,
    PRIMARY KEY (topic_name, partition_id)
);

-- Brokers
CREATE TABLE brokers (
    broker_id INT PRIMARY KEY,
    host VARCHAR(255) NOT NULL,
    port INT NOT NULL,
    rack VARCHAR(100),
    endpoints JSONB,
    registered_at TIMESTAMP DEFAULT NOW(),
    last_heartbeat TIMESTAMP
);

-- Consumer Groups (stored in __consumer_offsets topic)
-- Offset Commit Record
{
    "group": "order-processor",
    "topic": "orders",
    "partition": 0,
    "offset": 12345,
    "metadata": "",
    "commit_timestamp": 1705320600000
}

-- Group Metadata Record
{
    "group": "order-processor",
    "generation": 5,
    "protocol": "range",
    "leader": "consumer-1-uuid",
    "members": [
        {
            "member_id": "consumer-1-uuid",
            "client_id": "consumer-1",
            "client_host": "10.0.0.1",
            "assignment": "base64-encoded-assignment"
        }
    ]
}
```

---

## 7. Configuration Schema

### Broker Configuration

```properties
# Broker identity
broker.id=1
listeners=PLAINTEXT://0.0.0.0:9092
advertised.listeners=PLAINTEXT://broker1.example.com:9092

# Log configuration
log.dirs=/data/kafka-logs
num.partitions=10
default.replication.factor=3
min.insync.replicas=2

# Log retention
log.retention.hours=168
log.retention.bytes=-1
log.segment.bytes=1073741824
log.cleanup.policy=delete

# Replication
replica.fetch.max.bytes=1048576
replica.fetch.wait.max.ms=500
replica.lag.time.max.ms=30000

# Network
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# Controller
controller.quorum.voters=1@controller1:9093,2@controller2:9093,3@controller3:9093
```

### Topic Configuration

```properties
# Retention
retention.ms=604800000           # 7 days
retention.bytes=-1               # Unlimited
segment.bytes=1073741824         # 1 GB segments
segment.ms=604800000             # Roll segment after 7 days

# Compaction
cleanup.policy=delete            # or "compact" or "compact,delete"
min.cleanable.dirty.ratio=0.5
delete.retention.ms=86400000

# Replication
min.insync.replicas=2

# Compression
compression.type=producer        # or lz4, snappy, zstd, gzip

# Message size
max.message.bytes=1048576        # 1 MB
```

---

## 8. Error Handling

### Error Code Reference

| Code | Name | Description |
|------|------|-------------|
| 0 | NONE | Success |
| 1 | OFFSET_OUT_OF_RANGE | Requested offset invalid |
| 3 | UNKNOWN_TOPIC_OR_PARTITION | Topic doesn't exist |
| 5 | LEADER_NOT_AVAILABLE | No leader for partition |
| 6 | NOT_LEADER_FOR_PARTITION | Broker is not leader |
| 7 | REQUEST_TIMED_OUT | Request timed out |
| 10 | MESSAGE_TOO_LARGE | Message exceeds max size |
| 19 | NOT_ENOUGH_REPLICAS | ISR too small |
| 22 | ILLEGAL_GENERATION | Consumer group rebalancing |
| 25 | UNKNOWN_MEMBER_ID | Consumer needs to rejoin |
| 27 | REBALANCE_IN_PROGRESS | Consumer group rebalancing |

### Retry Strategy

```java
public class RetryPolicy {
    
    // Retryable errors
    private static final Set<Short> RETRYABLE = Set.of(
        ErrorCode.LEADER_NOT_AVAILABLE,
        ErrorCode.NOT_LEADER_FOR_PARTITION,
        ErrorCode.REQUEST_TIMED_OUT,
        ErrorCode.NOT_ENOUGH_REPLICAS,
        ErrorCode.NOT_ENOUGH_REPLICAS_AFTER_APPEND
    );
    
    public boolean shouldRetry(short errorCode) {
        return RETRYABLE.contains(errorCode);
    }
    
    public long getBackoffMs(int attempt) {
        // Exponential backoff: 100ms, 200ms, 400ms, ...
        return Math.min(100L * (1L << attempt), 10000L);
    }
}
```

