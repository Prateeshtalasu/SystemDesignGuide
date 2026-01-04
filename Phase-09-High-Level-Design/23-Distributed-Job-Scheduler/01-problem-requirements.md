# Distributed Job Scheduler (Cron at Scale) - Problem & Requirements

## What is a Distributed Job Scheduler?

A distributed job scheduler is a system that schedules, executes, and monitors jobs across multiple machines. It handles job submission, scheduling algorithms, execution on worker pools, retry logic, and job dependencies (DAG).

**Example Flow:**
```
User submits job → Job queued → Scheduler assigns to worker → 
Worker executes → Job completes → Results stored
```

### Why Does This Exist?

1. **Task Automation**: Run periodic tasks (data processing, reports)
2. **Resource Management**: Distribute work across machines
3. **Reliability**: Retry failed jobs, handle failures
4. **Dependency Management**: Execute jobs in order (DAG)
5. **Scalability**: Handle millions of jobs

### What Breaks Without It?

- Manual job execution required
- No centralized scheduling
- Jobs run on single machine (bottleneck)
- No retry logic for failures
- No dependency management

---

## Clarifying Questions (Ask the Interviewer)

| Question | Why It Matters | Assumed Answer |
|----------|----------------|----------------|
| What's the scale (jobs per day)? | Determines infrastructure size | 100 million jobs/day |
| What types of jobs? | Affects execution model | Long-running, batch, periodic |
| Job dependencies? | Affects scheduling complexity | Yes, DAG support |
| Job priority? | Affects scheduling algorithm | Yes, priority levels |
| Job retry policy? | Affects failure handling | Exponential backoff, max 3 retries |
| Job timeout? | Affects resource management | Configurable per job |

---

## Functional Requirements

### Core Features (Must Have)

1. **Job Submission**
   - Submit jobs with schedule (cron, one-time, recurring)
   - Define job metadata (name, priority, timeout)
   - Support job dependencies (DAG)

2. **Job Scheduling**
   - Schedule jobs based on cron expression
   - Priority-based scheduling
   - Dependency resolution (execute in order)

3. **Job Execution**
   - Assign jobs to workers
   - Execute jobs on worker pool
   - Monitor job execution

4. **Retry Logic**
   - Retry failed jobs
   - Exponential backoff
   - Max retry limit

5. **Job Monitoring**
   - Track job status (pending, running, completed, failed)
   - View job history
   - Job metrics and logs

6. **Worker Management**
   - Register workers
   - Health checks
   - Load balancing

### Secondary Features (Nice to Have)

7. **Job Pausing/Resuming**
   - Pause job execution
   - Resume paused jobs

8. **Job Cancellation**
   - Cancel running jobs
   - Cleanup resources

9. **Job Notifications**
   - Notify on job completion/failure
   - Email/SMS alerts

---

## Non-Functional Requirements

### Performance

| Metric | Target | Rationale |
|--------|--------|-----------|
| Job submission latency | < 100ms (p95) | Fast job submission |
| Job scheduling latency | < 50ms | Quick job assignment |
| Job execution overhead | < 1% | Minimal scheduler overhead |

### Scale

| Metric | Value | Calculation |
|--------|-------|-------------|
| Jobs/day | 100 million | Given assumption |
| Jobs/second (peak) | 5,000 | 100M / 86,400 × 50x peak |
| Concurrent jobs | 100,000 | Multiple jobs running |
| Workers | 10,000 | Distributed execution |

### Reliability

- **Availability**: 99.99% (52 minutes downtime/year)
- **Durability**: 99.999999999% (11 nines)
- **Consistency**: Eventual consistency for job state (acceptable delay)

### Security

- **Authentication**: Job submission requires auth
- **Authorization**: Role-based access control
- **Isolation**: Jobs run in isolated environments

---

## What's Out of Scope

1. **Job Code Execution**: Actual job code (assume separate execution)
2. **Resource Provisioning**: Worker provisioning (assume separate service)
3. **Job Results Storage**: Result storage (assume separate service)
4. **Job Templates**: Pre-defined job templates

---

## System Constraints

### Technical Constraints

1. **Job Scheduling**: Must handle millions of scheduled jobs
   - Efficient cron evaluation
   - Priority queue management

2. **Worker Capacity**: Limited worker resources
   - Load balancing across workers
   - Worker health monitoring

3. **Dependency Resolution**: Complex DAG execution
   - Topological sort for dependencies
   - Parallel execution where possible

---

## Success Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Job success rate | > 99% | Successful jobs / Total jobs |
| Job scheduling latency | < 50ms | Time to assign job to worker |
| Worker utilization | > 80% | Active workers / Total workers |
| Job retry rate | < 5% | Retried jobs / Total jobs |

---

## User Stories

### Story 1: Submit and Execute Job

```
As a user,
I want to submit a job and have it executed,
So that my task runs automatically.

Acceptance Criteria:
- Submit job with schedule
- Job scheduled and executed
- Job status tracked
- Results available
```

### Story 2: Handle Job Dependencies

```
As a system,
I want to execute jobs in dependency order,
So that dependent jobs run after prerequisites.

Acceptance Criteria:
- Define job dependencies (DAG)
- Execute jobs in topological order
- Handle dependency failures
```

---

## Core Components Overview

### 1. Scheduler Service
- Manages job scheduling
- Cron evaluation
- Priority queue management

### 2. Job Queue Service
- Manages job queues
- Job assignment to workers
- Load balancing

### 3. Worker Service
- Executes jobs
- Reports job status
- Health checks

### 4. Dependency Resolver
- Resolves job dependencies
- Topological sort
- Parallel execution

---

## Summary

| Aspect | Decision |
|-------|----------|
| Primary use case | Schedule and execute jobs at scale |
| Scale | 100 million jobs/day, 5,000 jobs/second peak |
| Key challenge | Efficient scheduling, dependency resolution |
| Architecture pattern | Distributed scheduler with worker pools |
| Consistency | Eventual for job state |
| Scheduling | Priority-based with cron support |

