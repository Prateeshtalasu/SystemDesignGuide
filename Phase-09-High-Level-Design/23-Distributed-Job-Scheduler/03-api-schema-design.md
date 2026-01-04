# Distributed Job Scheduler - API & Schema Design

## API Design Philosophy

Job scheduler APIs prioritize:

1. **Reliability**: Idempotent job submission
2. **Performance**: Fast job submission and status queries
3. **Flexibility**: Support various job types and schedules
4. **Monitoring**: Comprehensive job status and metrics

---

## Base URL Structure

```
REST API:     https://api.scheduler.com/v1
WebSocket:    wss://ws.scheduler.com/v1
```

---

## API Versioning Strategy

URL path versioning (`/v1/`, `/v2/`) for backward compatibility.

---

## Rate Limiting Headers

```http
X-RateLimit-Limit: 10000
X-RateLimit-Remaining: 9999
X-RateLimit-Reset: 1640000000
```

**Rate Limits:**

| Endpoint Type | Requests/minute |
|---------------|-----------------|
| Job Submission | 1000 |
| Job Status Query | 10000 |
| Worker Heartbeat | 60000 |

---

## Error Model

Standard error envelope structure:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "field": "field_name",
      "reason": "Specific reason"
    },
    "request_id": "req_123456"
  }
}
```

**Error Codes:**

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | INVALID_INPUT | Request validation failed |
| 400 | INVALID_CRON | Invalid cron expression |
| 400 | INVALID_DEPENDENCY | Circular dependency detected |
| 404 | JOB_NOT_FOUND | Job not found |
| 409 | JOB_EXISTS | Job already exists |
| 429 | RATE_LIMITED | Rate limit exceeded |

---

## Idempotency Implementation

**Idempotency-Key Header:**

```http
Idempotency-Key: <uuid-v4>
```

**Per-Endpoint Idempotency:**

| Endpoint | Idempotent? | Mechanism |
|----------|-------------|-----------|
| POST /v1/jobs | Yes | Idempotency-Key header |
| PUT /v1/jobs/{id} | Yes | Idempotency-Key header |
| POST /v1/jobs/{id}/cancel | Yes | Safe to retry |

---

## Core API Endpoints

### 1. Job Management

#### Submit Job

**Endpoint:** `POST /v1/jobs`

**Request:**

```json
{
  "name": "daily-report",
  "schedule": {
    "type": "cron",
    "expression": "0 0 * * *"
  },
  "priority": 5,
  "timeout_seconds": 3600,
  "retry_policy": {
    "max_retries": 3,
    "backoff": "exponential",
    "initial_delay_seconds": 60
  },
  "dependencies": ["job_123", "job_456"]
}
```

**Response (201 Created):**

```json
{
  "job": {
    "job_id": "job_abc123",
    "name": "daily-report",
    "status": "scheduled",
    "schedule": {
      "type": "cron",
      "expression": "0 0 * * *",
      "next_run": "2024-01-21T00:00:00Z"
    },
    "priority": 5,
    "created_at": "2024-01-20T10:00:00Z"
  }
}
```

#### Get Job Status

**Endpoint:** `GET /v1/jobs/{job_id}`

**Response (200 OK):**

```json
{
  "job": {
    "job_id": "job_abc123",
    "name": "daily-report",
    "status": "running",
    "schedule": {...},
    "current_run": {
      "run_id": "run_xyz789",
      "started_at": "2024-01-20T00:00:00Z",
      "worker_id": "worker_123"
    },
    "last_run": {
      "run_id": "run_prev456",
      "status": "completed",
      "completed_at": "2024-01-19T00:30:00Z"
    },
    "retry_count": 0
  }
}
```

#### Cancel Job

**Endpoint:** `POST /v1/jobs/{job_id}/cancel`

**Response (200 OK):**

```json
{
  "job_id": "job_abc123",
  "status": "cancelled",
  "cancelled_at": "2024-01-20T10:00:00Z"
}
```

---

### 2. Worker Management

#### Register Worker

**Endpoint:** `POST /v1/workers`

**Request:**

```json
{
  "worker_id": "worker_123",
  "capacity": 10,
  "capabilities": ["python", "java", "shell"]
}
```

**Response (200 OK):**

```json
{
  "worker": {
    "worker_id": "worker_123",
    "status": "active",
    "capacity": 10,
    "current_jobs": 5,
    "registered_at": "2024-01-20T10:00:00Z"
  }
}
```

#### Worker Heartbeat

**Endpoint:** `POST /v1/workers/{worker_id}/heartbeat`

**Request:**

```json
{
  "status": "healthy",
  "current_jobs": 5,
  "capacity": 10
}
```

---

## Database Schema Design

### Jobs Schema

```sql
CREATE TABLE jobs (
    job_id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    user_id VARCHAR(50) NOT NULL,
    
    schedule_type VARCHAR(20) NOT NULL,  -- cron, one_time, recurring
    cron_expression VARCHAR(100),
    next_run_at TIMESTAMP WITH TIME ZONE,
    
    priority INTEGER DEFAULT 5,
    timeout_seconds INTEGER,
    
    status VARCHAR(20) DEFAULT 'scheduled',
    
    retry_policy JSONB,
    metadata JSONB,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CONSTRAINT valid_status CHECK (status IN ('scheduled', 'queued', 'running', 'completed', 'failed', 'cancelled'))
);

CREATE INDEX idx_jobs_status ON jobs(status);
CREATE INDEX idx_jobs_next_run ON jobs(next_run_at) WHERE status = 'scheduled';
CREATE INDEX idx_jobs_user ON jobs(user_id);
```

### Job Dependencies Schema

```sql
CREATE TABLE job_dependencies (
    id BIGSERIAL PRIMARY KEY,
    job_id VARCHAR(50) NOT NULL,
    depends_on_job_id VARCHAR(50) NOT NULL,
    
    CONSTRAINT fk_job FOREIGN KEY (job_id) REFERENCES jobs(job_id) ON DELETE CASCADE,
    CONSTRAINT fk_depends_on FOREIGN KEY (depends_on_job_id) REFERENCES jobs(job_id) ON DELETE CASCADE,
    UNIQUE(job_id, depends_on_job_id)
);

CREATE INDEX idx_job_dependencies_job ON job_dependencies(job_id);
CREATE INDEX idx_job_dependencies_depends ON job_dependencies(depends_on_job_id);
```

### Job Runs Schema

```sql
CREATE TABLE job_runs (
    run_id VARCHAR(50) PRIMARY KEY,
    job_id VARCHAR(50) NOT NULL,
    worker_id VARCHAR(50),
    
    status VARCHAR(20) DEFAULT 'pending',
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    failed_at TIMESTAMP WITH TIME ZONE,
    
    retry_count INTEGER DEFAULT 0,
    error_message TEXT,
    
    CONSTRAINT fk_job FOREIGN KEY (job_id) REFERENCES jobs(job_id) ON DELETE CASCADE,
    CONSTRAINT valid_status CHECK (status IN ('pending', 'running', 'completed', 'failed', 'cancelled'))
);

CREATE INDEX idx_job_runs_job ON job_runs(job_id, started_at DESC);
CREATE INDEX idx_job_runs_status ON job_runs(status);
CREATE INDEX idx_job_runs_worker ON job_runs(worker_id);
```

### Workers Schema

```sql
CREATE TABLE workers (
    worker_id VARCHAR(50) PRIMARY KEY,
    status VARCHAR(20) DEFAULT 'active',
    capacity INTEGER NOT NULL,
    current_jobs INTEGER DEFAULT 0,
    
    last_heartbeat TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    registered_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CONSTRAINT valid_status CHECK (status IN ('active', 'inactive', 'dead'))
);

CREATE INDEX idx_workers_status ON workers(status);
CREATE INDEX idx_workers_heartbeat ON workers(last_heartbeat);
```

---

## Summary

| Component | Technology/Approach |
|-----------|-------------------|
| API Versioning | URL path versioning (/v1/) |
| Authentication | JWT tokens |
| Idempotency | Idempotency-Key header + Redis |
| Job Storage | PostgreSQL |
| Job Queue | Redis (priority queue) |
| Worker Management | PostgreSQL + Heartbeat |

