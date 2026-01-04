# Distributed Job Scheduler - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, failure scenarios, and level-specific expectations.

---

## Trade-off Questions

### Q1: How do you handle job scheduling at scale?

**Answer:**

**Problem:** Millions of scheduled jobs, need efficient cron evaluation

**Solution: Priority Queue + Time-based Indexing**

```java
@Service
public class JobScheduler {
    
    public void scheduleJob(Job job) {
        // 1. Calculate next run time
        Instant nextRun = cronEvaluator.getNextRun(job.getCronExpression());
        job.setNextRunAt(nextRun);
        
        // 2. Add to time-based index
        String timeKey = "jobs:" + nextRun.toEpochMilli();
        redis.opsForSet().add(timeKey, job.getJobId());
        
        // 3. Add to priority queue
        long score = job.getPriority() * 1000000L + nextRun.toEpochMilli();
        redis.opsForZSet().add("job_queue", job.getJobId(), score);
    }
    
    @Scheduled(fixedRate = 1000)  // Every second
    public void processScheduledJobs() {
        long currentTime = Instant.now().toEpochMilli();
        
        // Get jobs scheduled for current time
        String timeKey = "jobs:" + currentTime;
        Set<String> jobIds = redis.opsForSet().members(timeKey);
        
        for (String jobId : jobIds) {
            // Move to execution queue
            moveToExecutionQueue(jobId);
            redis.opsForSet().remove(timeKey, jobId);
        }
    }
}
```

**Benefits:**
- Efficient cron evaluation (only check current time)
- Priority-based execution
- Scalable to millions of jobs

---

### Q2: How do you prevent duplicate job execution?

**Answer:**

**Problem:** Same job executed multiple times (race condition)

**Solution: Distributed Locking + Idempotency**

```java
@Service
public class JobExecutionService {
    
    public ExecutionResult executeJob(String jobId) {
        String lockKey = "job_lock:" + jobId;
        
        // Acquire distributed lock
        Lock lock = distributedLock.acquire(lockKey, Duration.ofMinutes(60));
        
        try {
            // Check if job already running
            Job job = jobRepository.findById(jobId);
            if (job.getStatus() == JobStatus.RUNNING) {
                return ExecutionResult.alreadyRunning();
            }
            
            // Mark as running
            job.setStatus(JobStatus.RUNNING);
            jobRepository.save(job);
            
            // Execute job
            JobResult result = jobExecutor.execute(job);
            
            // Update status
            job.setStatus(result.isSuccess() ? JobStatus.COMPLETED : JobStatus.FAILED);
            jobRepository.save(job);
            
            return ExecutionResult.success(result);
            
        } finally {
            lock.release();
        }
    }
}
```

**Additional Safeguards:**
- Database unique constraint on (job_id, status) where status = 'running'
- Idempotency key for job submission
- Worker assignment (only one worker per job)

---

### Q3: How do you handle job dependencies (DAG)?

**Answer:**

**Problem:** Jobs with dependencies must execute in order

**Solution: Topological Sort + Dependency Tracking**

```java
@Service
public class DependencyResolver {
    
    public List<String> resolveDependencies(String jobId) {
        // 1. Build dependency graph
        Map<String, List<String>> graph = buildDependencyGraph(jobId);
        
        // 2. Topological sort
        List<String> executionOrder = topologicalSort(graph);
        
        // 3. Check if dependencies completed
        for (String depJobId : executionOrder) {
            Job depJob = jobRepository.findById(depJobId);
            if (depJob.getStatus() != JobStatus.COMPLETED) {
                return Collections.emptyList();  // Dependencies not ready
            }
        }
        
        return executionOrder;  // All dependencies ready
    }
    
    private List<String> topologicalSort(Map<String, List<String>> graph) {
        // Kahn's algorithm for topological sort
        // Returns jobs in dependency order
    }
}
```

---

## Scaling Questions

### Q4: How would you scale from 100M to 1B jobs/day?

**Answer:**

**Current State (100M jobs/day):**
- 15 scheduler servers
- 10,000 workers
- 100,000 jobs/second peak

**Scaling to 1B jobs/day:**

1. **Horizontal Scaling**
   ```
   Schedulers: 15 → 150 (10x)
   Workers: 10,000 → 100,000 (10x)
   ```

2. **Database Sharding**
   ```
   Jobs table: Shard by user_id hash
   10 shards → 100 shards
   ```

3. **Queue Partitioning**
   ```
   Job queue: Partition by priority
   1 queue → 10 queues (one per priority level)
   ```

4. **Worker Specialization**
   ```
   Specialized worker pools per job type
   Better resource utilization
   ```

**Bottlenecks:**
- Cron evaluation (time-based indexing)
- Queue depth (partition queues)
- Worker capacity (auto-scaling)

---

## Failure Scenarios

### Scenario 1: Worker Dies During Job Execution

**Problem:** Worker crashes, job status unknown

**Solution: Heartbeat + Timeout**

```java
@Service
public class WorkerHealthMonitor {
    
    @Scheduled(fixedRate = 30000)  // Every 30 seconds
    public void checkWorkerHealth() {
        List<Worker> workers = workerRepository.findActiveWorkers();
        
        for (Worker worker : workers) {
            Duration sinceLastHeartbeat = Duration.between(
                worker.getLastHeartbeat(), Instant.now()
            );
            
            if (sinceLastHeartbeat.toMinutes() > 5) {
                // Worker dead, reassign jobs
                reassignJobs(worker);
                worker.setStatus(WorkerStatus.DEAD);
                workerRepository.save(worker);
            }
        }
    }
    
    private void reassignJobs(Worker deadWorker) {
        List<Job> runningJobs = jobRepository.findByWorkerId(deadWorker.getWorkerId());
        
        for (Job job : runningJobs) {
            // Mark job as failed, allow retry
            job.setStatus(JobStatus.FAILED);
            job.setRetryCount(job.getRetryCount() + 1);
            jobRepository.save(job);
            
            // Re-queue for retry
            if (job.getRetryCount() < job.getMaxRetries()) {
                jobQueueService.enqueueJob(job);
            }
        }
    }
}
```

---

### Scenario 2: Scheduler Service Down

**Problem:** Cannot schedule new jobs

**Solution:**
1. Multiple scheduler instances (redundancy)
2. Health check-based failover
3. Jobs continue executing (workers independent)
4. New job submissions queued for when scheduler recovers

---

## Level-Specific Expectations

### L4 (Entry-Level)

**What's Expected:**
- Basic job scheduling understanding
- Simple queue-based design
- Basic retry logic

---

### L5 (Mid-Level)

**What's Expected:**
- Priority queue understanding
- Dependency resolution
- Worker management
- Failure handling

---

### L6 (Senior)

**What's Expected:**
- System evolution (MVP → Production → Scale)
- Efficient cron evaluation
- DAG execution optimization
- Cost optimization

---

## Common Interviewer Pushbacks

### "Your scheduler is too complex"

**Response:**
"For MVP, a simple queue with cron evaluation works. As we scale, we need time-based indexing and priority queues. The key is designing for evolution."

---

### "What if cron evaluation is slow?"

**Response:**
"We'd use time-based indexing to only check jobs scheduled for current time. We'd also cache cron expressions and pre-compute next run times."

---

## Summary

| Question Type | Key Points to Cover |
|--------------|---------------------|
| Trade-offs | Priority queue vs FIFO, time-based indexing |
| Scaling | Horizontal scaling, queue partitioning |
| Failures | Worker health monitoring, job reassignment |
| Edge cases | Dependency resolution, duplicate execution |
| Evolution | MVP → Production → Scale phases |

