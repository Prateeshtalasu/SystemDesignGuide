Design the system: <HLD PROBLEM NAME>

FOLLOW THESE STEPS EXACTLY:

STEP 1: REQUIREMENTS & SCOPE

- Functional requirements (what the system does)
- Non-functional requirements (scale, latency, availability, durability targets)
  - Include p50/p95/p99 latency goals
  - Include availability target (99.9/99.99) and what it applies to
  - Include durability / data-loss tolerance
- Clear assumptions stated explicitly (numbers + rationale)
- What is explicitly OUT of scope
- Clarifying questions you would ask the interviewer
- Success metrics (SLOs + product metrics)
- User stories (at least 2) with acceptance criteria

STEP 2: CAPACITY ESTIMATION (Back-of-Envelope)

- Daily/Monthly active users
- Read vs Write ratio
- QPS (Queries Per Second) for each operation
  - Show mental math shortcuts
  - Peak vs average traffic calculations
  - Geographic distribution considerations
- Storage requirements (calculate step by step)
  - Show assumptions clearly
  - Include growth projections (1 year, 5 years)
  - Include replication + index/storage overhead
- Bandwidth requirements (incoming vs outgoing, peak calculations)
- Memory requirements (cache sizing)
  - Working set estimate + cache hit-rate target
- Traffic growth projections
- Show all math, no magic numbers
- Include common FAANG interview numbers for reference

STEP 3: API DESIGN

- RESTful endpoints with HTTP methods
- Request/response formats (JSON examples)
- Pagination strategy (cursor vs offset) and why
- Input validation approach
- API versioning strategy
- Backward compatibility considerations
- Rate limiting headers + quota model (per user, per IP, per API key)
- Authentication/Authorization headers
- Idempotency strategy for writes (idempotency keys, retry-safe semantics)
- Error model (standard error envelope, HTTP codes, retryable vs non-retryable)

STEP 4: DATA MODEL & STORAGE

- Database choice with detailed justification
- Complete schema design with all fields
- Relationships and foreign keys (or justify why not)
- Indexes and why each exists
- Consistency model (strong vs eventual)
  - When to use strong consistency
  - When to use eventual consistency
  - Read-after-write consistency problems and solutions
  - Causal consistency if applicable
  - Session consistency if applicable
- Sharding strategy (shard key selection, reasoning)
  - Hot partition problem and solutions
  - Resharding strategy
- Replication strategy
  - Leader-follower vs multi-leader vs leaderless
  - Sync vs async replication trade-offs
- Data retention and archival
- Backup and recovery strategy (explicit RPO/RTO targets)

STEP 5: ARCHITECTURE (DATA MODEL + DIAGRAMS)

- First list every component and explain why it exists
- Clear boundaries between services
- Synchronous vs asynchronous paths marked
- Provide the architecture diagram:
  - ASCII diagram for simple systems OR Mermaid syntax for complex systems
- Request flow with numbered steps (at least 2 core flows)
- Data flow (separate from request flow if different)
- Failure points identified and where the system degrades gracefully

STEP 6: ASYNC, MESSAGING & EVENT FLOW (IF APPLICABLE)
Only include if the system requires async processing. If not, explicitly state why.

For EACH messaging technology used (Kafka/RabbitMQ/SQS/PubSub), you MUST write in this order:

A) CONCEPT (WHAT IT IS)

- What is this system (1-2 paragraphs, no hand-waving)
- What problems it solves (buffering, decoupling, ordering, durability)

B) OUR USAGE (HOW WE USE IT HERE)

- Why async vs sync for THIS problem
- What events exist and why
- Topic/queue design and partition strategy
- Partition key choice and reasoning (include hot partition risk)
- Consumer group design
- Ordering guarantees required (global vs per key)
- Offset management and replay strategy
- Deduplication strategy (idempotent consumers)
- Outbox pattern / CDC (Debezium) if strong DB + async events are both involved

C) REAL STEP-BY-STEP SIMULATION (MANDATORY)
Write a concrete pipeline like:

"User does X → Service A receives request → DB transaction → Outbox row → CDC reads → Kafka topic → partition chosen by <key> → Consumers process → offsets committed → side effects"

Then you MUST answer:

- What if consumer crashes mid-batch?
- What if duplicate events occur?
- What if a partition becomes hot?
- How does retry work (backoff, DLQ)?
- What breaks first and what degrades gracefully?

If NOT applicable, state: "This system is synchronous because..." and explain what would change if async were introduced.

STEP 7: CACHING (REDIS) + CACHE FLOW SIMULATION
If Redis (or any cache) is used, you MUST write in this order:

A) CONCEPT (WHAT IT IS)

- What Redis is and what caching pattern means (cache-aside, write-through, write-back)
- What consistency tradeoff we accept

B) OUR USAGE (HOW WE USE IT HERE)

- Exact keys and values
- Who reads Redis, who writes Redis
- TTL behavior and why TTL value is chosen
- Eviction policy and why
- Invalidation strategy (on update/delete)
- Why Redis vs local cache vs DB indexes

C) REAL STEP-BY-STEP SIMULATION (MANDATORY)
Include BOTH read and write paths:

1. Cache HIT path:
   Request → Service → Redis HIT → response (latency target)

2. Cache MISS path:
   Request → Service → Redis MISS → DB query → store in Redis with TTL → response

Then you MUST answer:

- What if Redis is down?
- What happens to DB load and latency?
- What is the circuit breaker behavior?
- Why this TTL?
- Why write-through vs write-back vs cache-aside?
- Negative caching (e.g., NOT_FOUND) and when to use it

If caching is NOT used, state why (and how you still meet latency).

STEP 8: SEARCH (ELASTICSEARCH / OPENSEARCH) + INVERTED INDEX SIMULATION (IF APPLICABLE)
Only include if the system has search requirements beyond primary-key lookups.

For EACH search technology used, you MUST write in this order:

A) CONCEPT (WHAT IT IS)

- What an inverted index is
- How term → posting list works at a high level
- What gets indexed (documents), not rows

B) OUR USAGE (HOW WE USE IT HERE)

- Source of truth (DB) and how we sync to search index
- Indexer service design (push vs pull)
- Mapping strategy (fields, analyzers, tokenization)
- Query patterns (term, prefix, fuzzy, filters)
- Ranking approach (BM25 or custom signals)

C) REAL STEP-BY-STEP SIMULATION (MANDATORY)
Data flow:
DB (or Kafka) → Indexer → transform to document → index → inverted index built

Query flow:
User query → analyzer/tokenizer → terms → posting lists → ranking → results

Then you MUST answer:

- How do we handle stale index vs DB truth?
- How do we reindex safely?
- What happens if indexer lags?
- Failure modes and recovery

If NOT applicable, state: "Search is not required because..." and explain what would change if search were added later.

STEP 9: SCALING & RELIABILITY

- Load balancing approach
- Rate limiting implementation details
- Circuit breaker placement (exact services)
- Timeout and retry policies (with jitter/backoff)
- Replication and failover
- Graceful degradation strategies (what to drop first under stress)
- Disaster recovery approach (multi-AZ, multi-region, active-active vs active-passive)
- What breaks first under load and why

STEP 10: MONITORING & OBSERVABILITY

- Key metrics (latency p50/p95/p99, error rate, throughput, saturation)
- Alerting thresholds and escalation (paging vs ticketing)
- Logging strategy (structured logs, redaction of PII)
- Distributed tracing (trace IDs, span boundaries)
- Health checks (liveness/readiness)
- Dashboards (golden signals + business KPIs)

STEP 11: SECURITY CONSIDERATIONS

- Authentication mechanism
- Authorization and access control
- Data encryption (at rest and in transit)
- Input validation and sanitization
- Rate limiting for abuse prevention
- PII handling and compliance (GDPR, deletion, retention)
- Security headers and HTTPS
- Threats and mitigations (enumeration, abuse, injection, replay)

STEP 12: SIMULATION (END-TO-END USER JOURNEYS + FAILURE/RECOVERY)
Walk through 1-2 complete user journeys:

For EACH journey:

- User action by action
- Request path step-by-step
- Data writes and reads step-by-step
- Async triggers (if any) step-by-step
- Cache interaction (hit/miss) step-by-step
- Search interaction (if any) step-by-step

MANDATORY: Failure & recovery walkthrough:

- Pick one realistic failure
- Show what degrades, what stays up
- Show what recovers automatically
- Show what requires human intervention
- Show how we prevent cascading failures (timeouts, circuit breakers, load shedding)

STEP 13: COST ANALYSIS

- Major cost drivers (storage, compute, network, managed services)
- Cost optimization strategies
- Build vs buy decisions for each component
- At what scale does this become expensive?
- Cost per user or per request estimate (show assumptions)

STEP 14: TRADEOFFS & INTERVIEW QUESTIONS (WITH ANSWERS)

- Why X database over Y?
- Why this caching strategy?
- Why async vs sync here?
- What breaks first under load?
- How does this system evolve over time?
  - MVP phase (what to build first)
  - Production phase (what to add)
  - Scale phase (how it changes at 10x, 100x)
  - Migration strategies between phases
- What would you do with unlimited budget?
- What would you cut if you had 2 weeks to build?
- Common interviewer pushbacks and responses
- Level-specific answers:
  - L4: Basic trade-offs, correctness, simple scaling
  - L5: Production concerns, deep dives into bottlenecks
  - L6: Multiple architectures, migrations, org-level trade-offs

STEP 15: FAILURE SCENARIOS & RESILIENCE (DETAILED)

- Single points of failure identification
- What happens if each major component fails?
- Recovery strategy for each component
- Data loss prevention mechanisms
- Partial system failure handling
- Cascading failure prevention
- Circuit breaker configuration details
- Disaster recovery plan (RTO/RPO) and validation drills

OUTPUT FILE STRUCTURE:
This prompt generates content for 7 files (6 + split 5A/5B):

- STEP 1 → `01-problem-requirements.md`
- STEP 2 → `02-capacity-estimation.md`
- STEP 3 → `03-api-schema-design.md`
- STEP 4-5 → `04-data-model-architecture.md`
- STEP 6-8 → `05A-production-deep-dives-core.md`
- STEP 9-13 → `05B-production-deep-dives-ops.md`
- STEP 14-15 → `06-interview-grilling.md`

RULES:

- Explain every technology choice before using it
- ALWAYS separate Concept vs Our Usage for Kafka/Redis/Search (mandatory)
- ALWAYS include step-by-step event and data flow simulations (mandatory)
- ALWAYS include failure and recovery flows (mandatory)
- No shallow diagrams without explanation
- No missing reasoning for any decision
- Numbers must be calculated, not assumed
- If a step doesn't apply, explicitly state why
- Keep each file self-contained, do not say “covered elsewhere”
