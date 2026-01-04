# Distributed Job Scheduler - Data Model & Architecture

## Component Overview

| Component | Purpose | Why It Exists |
|-----------|---------|---------------|
| **Scheduler Service** | Manages job scheduling | Cron evaluation, priority queue |
| **Job Queue Service** | Manages job queues | Job assignment to workers |
| **Worker Service** | Executes jobs | Distributed job execution |
| **Dependency Resolver** | Resolves job dependencies | DAG execution order |

---

## Database Choices

| Data Type | Database | Rationale |
|-----------|----------|-----------|
| Jobs | PostgreSQL | ACID transactions, complex queries |
| Job Queue | Redis (priority queue) | Fast job assignment |
| Job Status | Redis (cache) + PostgreSQL | Real-time status, persistence |
| Workers | PostgreSQL | Worker management |
| Job Dependencies | PostgreSQL | DAG storage |

---

## Consistency Model

**CAP Theorem Tradeoff:**

We choose **Availability + Partition Tolerance (AP)**:
- **Availability**: Scheduler must always accept jobs
- **Partition Tolerance**: System continues during partitions
- **Consistency**: Sacrificed (job status may be stale briefly)

**Why AP over CP?**
- Job scheduling doesn't need strict consistency
- Better to accept jobs than fail completely
- Eventual consistency acceptable for job status

**ACID (Strong Consistency) for:**
- Job submission (PostgreSQL, prevent duplicates)
- Job dependencies (PostgreSQL, prevent circular dependencies)

**BASE (Eventual Consistency) for:**
- Job status updates (eventual synchronization)
- Worker capacity (acceptable staleness)

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    CLIENTS                                           │
│                    (API Clients, Web UI)                                            │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                 API GATEWAY                                          │
│                          (Load Balancer, Rate Limiting)                             │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
                    ▼                   ▼                   ▼
┌──────────────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   Scheduler Service      │  │  Job Queue       │  │  Worker Service  │
│  - Schedule jobs         │  │  Service         │  │  - Execute jobs  │
│  - Evaluate cron         │  │  - Assign jobs   │  │  - Report status │
│  - Priority queue        │  │  - Load balance  │  │  - Health checks │
└──────────────────────────┘  └──────────────────┘  └──────────────────┘
         │                              │                      │
         │                              │                      │
         └──────────────┬───────────────┼──────────────────────┘
                        │               │                      │
                        ▼               ▼                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                  DATA LAYER                                         │
│                                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Redis      │  │  PostgreSQL  │  │     Kafka    │  │   Workers    │          │
│  │  (Job Queue) │  │  (Jobs/      │  │  (Events)    │  │  (10,000)    │          │
│  │              │  │   Workers)   │  │              │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Job Scheduling Flow

```
Step 1: Job Submitted
User → Scheduler Service: Submit job with cron "0 0 * * *"
  │
  ▼
Step 2: Store Job
Scheduler Service → PostgreSQL: INSERT INTO jobs
  │
  ▼
Step 3: Evaluate Cron
Scheduler Service: Calculate next_run_at = "2024-01-21T00:00:00Z"
  │
  ▼
Step 4: Add to Priority Queue
Scheduler Service → Redis: ZADD job_queue <priority_score> job_id
  │
  ▼
Step 5: Schedule Execution
When next_run_at arrives:
  - Remove from queue
  - Assign to worker
  - Execute job
```

---

## Dependency Resolution Flow

```
Step 1: Job with Dependencies
Job A depends on Job B and Job C
  │
  ▼
Step 2: Check Dependencies
Dependency Resolver: Check if B and C completed
  │
  ▼
Step 3: Wait for Dependencies
If B or C not completed:
  - Job A remains in queue
  - Wait for dependencies
  │
  ▼
Step 4: Execute When Ready
When B and C completed:
  - Job A ready to execute
  - Assign to worker
  - Execute job
```

---

## Summary

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Job Storage | PostgreSQL | ACID transactions |
| Job Queue | Redis (priority queue) | Fast job assignment |
| Consistency | Eventual for status | Acceptable staleness |
| Scheduling | Priority-based + cron | Efficient scheduling |
| Workers | Distributed pool | Scalable execution |

