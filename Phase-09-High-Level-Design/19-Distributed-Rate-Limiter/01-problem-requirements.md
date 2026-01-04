# Distributed Rate Limiter - Problem & Requirements

## What is a Distributed Rate Limiter?

A Distributed Rate Limiter controls the rate of requests across multiple servers, preventing abuse and ensuring fair resource usage. It must work consistently across a distributed system where requests can hit any server.

**Example:**
```
User makes 100 requests/second
  ↓
Rate limiter checks: 1000 requests/minute limit
  ↓
First 1000 requests: Allowed
  ↓
Request 1001: Rate limited (429)
```

### Why Does This Exist?

1. **API Protection**: Prevent abuse and DoS attacks
2. **Fair Usage**: Ensure fair resource distribution
3. **Cost Control**: Limit expensive operations
4. **Compliance**: Meet SLA requirements

---

## Functional Requirements

### Core Features

1. **Token Bucket Algorithm**
   - Configurable rate limits
   - Burst handling
   - Per-user, per-IP, per-API limits

2. **Sliding Window**
   - Accurate rate limiting
   - Redis-based implementation
   - Distributed synchronization

3. **Multiple Strategies**
   - Fixed window
   - Sliding window
   - Token bucket

---

## Non-Functional Requirements

| Metric | Target |
|--------|--------|
| Latency | < 10ms p95 |
| Accuracy | 99.9% |
| Availability | 99.99% |
| Scale | 100K QPS |

---

## Summary

| Aspect | Decision |
|--------|----------|
| Algorithm | Token bucket + Sliding window |
| Storage | Redis |
| Scale | 100K QPS |


