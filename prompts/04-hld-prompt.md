# 🔹 PROMPT 4: HLD PROMPT (HIGH-LEVEL DESIGN)

> **Usage:** Use this for distributed system design problems.  
> **Examples:** URL Shortener, Twitter Feed, Uber, Google Drive, YouTube, Search Engine, etc.

---

```
Design the system: <HLD PROBLEM NAME>

FOLLOW THESE STEPS EXACTLY:

STEP 1: REQUIREMENTS & SCOPE
- Functional requirements (what the system does)
- Non-functional requirements (scale, latency, availability targets)
- Clear assumptions stated explicitly
- What is explicitly OUT of scope
- Clarifying questions you would ask the interviewer

STEP 2: CAPACITY ESTIMATION (Back-of-Envelope)
- Daily/Monthly active users
- Read vs Write ratio
- QPS (Queries Per Second) for each operation
  * Show mental math shortcuts (e.g., 1M users = ~100 QPS peak)
  * Peak vs average traffic calculations
  * Geographic distribution considerations
- Storage requirements (calculate step by step)
  * Show assumptions clearly
  * Include growth projections (1 year, 5 years)
- Bandwidth requirements
  * Incoming vs outgoing traffic
  * Peak bandwidth calculations
- Memory requirements (cache sizing)
- Traffic growth projections
- Show all math, no magic numbers
- Include common FAANG interview numbers for reference

STEP 3: API DESIGN
- RESTful endpoints with HTTP methods
- Request/response formats (JSON examples)
- Pagination strategy
- Input validation approach
- API versioning strategy
- Backward compatibility considerations
- Rate limiting headers
- Authentication/Authorization headers

STEP 4: DATA MODEL & STORAGE
- Database choice with detailed justification
- Complete schema design with all fields
- Relationships and foreign keys
- Indexes and why each exists
- Consistency model (strong vs eventual)
  * When to use strong consistency (financial transactions, critical data)
  * When to use eventual consistency (social feeds, analytics)
  * Read-after-write consistency problem and solutions
  * Causal consistency if applicable
  * Session consistency if applicable
- Sharding strategy (shard key selection, reasoning)
  * Hot partition problem and solutions
  * Resharding strategy
- Replication strategy
  * Leader-follower vs multi-leader vs leaderless
  * Sync vs async replication trade-offs
- Data retention and archival
- Backup and recovery strategy

STEP 5: ARCHITECTURE DIAGRAM
- ASCII diagram for simple systems OR Mermaid syntax for complex
- Every component labeled and explained BEFORE the diagram
- Clear boundaries between services
- Synchronous vs asynchronous paths marked
- Request flow with numbered steps
- Data flow (separate from request flow if different)
- Failure points identified

STEP 6: ASYNC, MESSAGING & CACHING (IF APPLICABLE)
Only include if the system requires async processing:
- Message broker choice (Kafka/RabbitMQ) with justification
- Topic design and partition strategy
- Consumer group design
- Offset management and replay strategy
- Outbox pattern / CDC if eventual consistency needed
- Redis usage: caching strategy, TTL, eviction policy
- ElasticSearch: indexing strategy if search needed
- If NOT applicable, state: "This system is synchronous because..."

STEP 7: SCALING & RELIABILITY
- Caching strategy (what, where, TTL, invalidation)
- Load balancing approach
- Rate limiting implementation
- Circuit breaker placement
- Timeout and retry policies
- Replication and failover
- Graceful degradation strategies
- Disaster recovery approach

STEP 8: MONITORING & OBSERVABILITY
- Key metrics to track (latency p50/p99, error rate, throughput)
- Alerting thresholds and escalation
- Logging strategy (what to log, structured logging format)
- Distributed tracing approach
- Health check endpoints
- Dashboard design (critical graphs)

STEP 9: SECURITY CONSIDERATIONS
- Authentication mechanism
- Authorization and access control
- Data encryption (at rest and in transit)
- Input validation and sanitization
- Rate limiting for abuse prevention
- PII handling and compliance (GDPR, etc.)
- Security headers and HTTPS

STEP 10: SIMULATION
Walk through 1-2 complete user journeys:
- User action by action
- Show how requests flow through the system
- Show where data is written and read
- Show async processes triggering
- Include a failure scenario with recovery

STEP 11: COST ANALYSIS
- Major cost drivers (storage, compute, network, managed services)
- Cost optimization strategies
- Build vs buy decisions for each component
- At what scale does this become expensive?
- Cost per user or per request estimate

STEP 12: TRADEOFFS & INTERVIEW QUESTIONS (WITH ANSWERS)
- Why X database over Y?
- Why this caching strategy?
- What breaks first under load?
- How does this system evolve over time?
  * MVP phase (what to build first)
  * Production phase (what to add)
  * Scale phase (how it changes at 10x, 100x)
  * Migration strategies between phases
- What would you do with unlimited budget?
- What would you cut if you had 2 weeks to build?
- Common interviewer pushbacks and responses
- Level-specific answers:
  * L4: Focus on basic trade-offs and understanding
  * L5: Deep dives into specific components, production experience
  * L6: Multiple solution approaches, complex trade-off analysis

STEP 13: FAILURE SCENARIOS & RESILIENCE
- Single points of failure identification
- What happens if [component] fails?
- Failure recovery strategies for each component
- Data loss prevention mechanisms
- Partial system failure handling
- Cascading failure prevention
- Circuit breaker placement and configuration
- Graceful degradation strategies
- Disaster recovery plan (RTO/RPO)

OUTPUT FILE STRUCTURE:
This prompt generates content for 6 files:
- STEP 1 → `01-problem-requirements.md`
- STEP 2 → `02-capacity-estimation.md`
- STEP 3 → `03-api-schema-design.md`
- STEP 5 → `04-architecture-diagrams.md`
- STEP 6-7, 9 → `05-tech-deep-dives.md`
- STEP 12-13 → `06-interview-grilling.md`

RULES:
- Explain every technology choice before using it
- Use real-world examples (how Uber, Netflix, Amazon solve this)
- No shallow diagrams without explanation
- No missing reasoning for any decision
- Numbers must be calculated, not assumed
- If a step doesn't apply, explicitly state why
```

