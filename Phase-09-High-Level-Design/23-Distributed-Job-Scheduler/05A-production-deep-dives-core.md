# Distributed Job Scheduler - Production Deep Dives (Core)

## Overview

This document covers core production components: asynchronous messaging (Kafka for job events), caching strategy (Redis for job queue and status), and priority queue management.

---

## 1. Async, Messaging & Event Flow (Kafka)

### A) CONCEPT: What is Kafka in Job Scheduler Context?

Kafka serves as the event backbone for:
1. **Job Events**: Stream job lifecycle (submitted, scheduled, running, completed, failed)
2. **Worker Events**: Stream worker status updates
3. **Dependency Events**: Stream dependency resolution events

### B) OUR USAGE: How We Use Kafka Here

**Topic Design:**

| Topic | Purpose | Partitions | Key |
|-------|---------|------------|-----|
| `job-events` | Job lifecycle | 20 | job_id |
| `worker-events` | Worker status | 10 | worker_id |
| `dependency-events` | Dependency resolution | 10 | job_id |

**Producer Configuration:**

```java
@Configuration
public class JobKafkaConfig {
    
    @Bean
    public ProducerFactory<String, JobEvent> jobEventProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Normal Flow: Job Execution Event**

```
Step 1: Job Scheduled
Scheduler Service: Job 'job_123' scheduled for execution
  │
  ▼
Step 2: Publish Event
Kafka Producer: job-events, partition 3, key 'job_123'
  Message: {"event_type": "job.scheduled", "job_id": "job_123"}
  │
  ▼
Step 3: Consumers Process
- Job Queue Service: Add to queue
- Dependency Resolver: Check dependencies
- Worker Service: Assign to worker
```

---

## 2. Caching (Redis)

### A) CONCEPT: What is Redis Caching in Scheduler Context?

Redis provides:
1. **Job Queue**: Priority queue for job assignment
2. **Job Status Cache**: Real-time job status
3. **Worker Status Cache**: Worker availability

### B) OUR USAGE: How We Use Redis Here

**Cache Types:**

| Cache | Key | TTL | Purpose |
|-------|-----|-----|---------|
| Job Queue | `job_queue` (sorted set) | None | Priority queue |
| Job Status | `job_status:{job_id}` | 1 hour | Current job status |
| Worker Status | `worker_status:{worker_id}` | 5 minutes | Worker availability |

**Priority Queue:**

```java
@Service
public class JobQueueService {
    
    public void enqueueJob(String jobId, int priority, Instant scheduledTime) {
        // Score = priority * 1000000 + timestamp (for ordering)
        long score = priority * 1000000L + scheduledTime.toEpochMilli();
        redis.opsForZSet().add("job_queue", jobId, score);
    }
    
    public String dequeueJob() {
        // Get highest priority job (lowest score = highest priority)
        Set<String> jobs = redis.opsForZSet().range("job_queue", 0, 0);
        if (jobs.isEmpty()) {
            return null;
        }
        String jobId = jobs.iterator().next();
        redis.opsForZSet().remove("job_queue", jobId);
        return jobId;
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Normal Flow: Job Queue Operations**

```
Step 1: Job Scheduled
Scheduler Service: Job 'job_123' scheduled, priority 5
  │
  ▼
Step 2: Add to Queue
Redis: ZADD job_queue 5000000 job_123
  Score = 5 * 1000000 + timestamp
  │
  ▼
Step 3: Worker Requests Job
Worker Service: Get next job from queue
  │
  ▼
Step 4: Dequeue Job
Redis: ZRANGE job_queue 0 0 (get highest priority)
Redis: ZREM job_queue job_123 (remove from queue)
  │
  ▼
Step 5: Assign to Worker
Worker Service: Assign job_123 to worker_456
```

---

## 3. Priority Queue Management

### A) CONCEPT: What is Priority Queue?

Priority queue orders jobs by priority, with higher priority jobs executed first.

**Why Needed:**
- Jobs have different priorities
- Critical jobs must execute before low-priority jobs
- Efficient job scheduling

### B) OUR USAGE: How We Use Priority Queue

**Implementation:**

```java
@Service
public class PriorityQueueManager {
    
    public void scheduleJob(Job job) {
        // Calculate priority score
        long score = calculatePriorityScore(job);
        
        // Add to Redis sorted set
        redis.opsForZSet().add("job_queue", job.getJobId(), score);
    }
    
    private long calculatePriorityScore(Job job) {
        // Higher priority = lower score (for min-heap behavior)
        int priority = job.getPriority();  // 1 (highest) to 10 (lowest)
        long timestamp = job.getScheduledTime().toEpochMilli();
        
        // Score = priority * 1000000 + timestamp
        // Lower score = higher priority
        return priority * 1000000L + timestamp;
    }
    
    public Job getNextJob() {
        // Get job with lowest score (highest priority)
        Set<String> jobIds = redis.opsForZSet().range("job_queue", 0, 0);
        if (jobIds.isEmpty()) {
            return null;
        }
        
        String jobId = jobIds.iterator().next();
        redis.opsForZSet().remove("job_queue", jobId);
        
        return jobRepository.findById(jobId);
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Priority Queue Operations:**

```
Queue State:
  job_1: priority 1, score 1000000
  job_2: priority 5, score 5000000
  job_3: priority 3, score 3000000

Step 1: Worker Requests Job
Worker: Get next job
  │
  ▼
Step 2: Get Highest Priority
Redis: ZRANGE job_queue 0 0
Result: job_1 (lowest score = highest priority)
  │
  ▼
Step 3: Remove from Queue
Redis: ZREM job_queue job_1
  │
  ▼
Step 4: Assign to Worker
Worker receives job_1 for execution
```

---

## Summary

| Component | Technology | Key Configuration |
|-----------|------------|-------------------|
| Event Streaming | Kafka | 20 partitions, job_id key |
| Job Queue | Redis (sorted set) | Priority-based ordering |
| Job Status Cache | Redis | 1-hour TTL |
| Priority Queue | Redis ZSET | Score = priority * 1000000 + timestamp |
| Worker Management | PostgreSQL + Heartbeat | Health checks every 30s |

