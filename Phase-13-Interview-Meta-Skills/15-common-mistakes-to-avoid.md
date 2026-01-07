# Common Mistakes to Avoid

## 0️⃣ Prerequisites

Before diving into common mistakes, you should understand:

- **Problem Approach Framework**: How to structure your interview (covered in Topic 1)
- **All Previous Topics**: Core system design knowledge and interview skills (covered in Topics 1-14)

Quick refresher: Knowing what NOT to do is as important as knowing what to do. Many candidates fail not because they lack knowledge, but because they make avoidable mistakes. Learning from others' mistakes is more efficient than making them yourself.

---

## 1️⃣ Why Mistakes Matter

System design interviews have a high signal-to-noise ratio. A few key mistakes can overshadow otherwise strong performance:

- **One major mistake** can change "hire" to "no hire"
- **Repeated small mistakes** accumulate into a negative impression
- **Easily avoidable mistakes** suggest lack of preparation
- **The same mistakes** are made by most candidates, making them predictable

---

## 2️⃣ The 15 Most Common Mistakes

### Mistake 1: Jumping to Solution Too Fast

**What it looks like**:
"Design Twitter. Okay, so we'll have a user service, a tweet service, a timeline service..."

**Specific examples**:
- **Example 1**: Interviewer says "Design a URL shortener" and candidate immediately starts: "We'll use a hash function to generate short URLs, store them in Redis..." without asking about scale, features, or constraints.
- **Example 2**: Interviewer says "Design a chat system" and candidate jumps to: "We'll use WebSockets, have a message service, user service..." without clarifying if it's 1-on-1 chat, group chat, or both.
- **Example 3**: Interviewer says "Design a search engine" and candidate starts designing distributed systems without asking if it's for web search, product search, or internal document search.

**Why it's bad**:
- Missed opportunity to clarify requirements
- May design for wrong scale or features
- Shows lack of senior-level thinking
- Interviewer wonders if you understand the problem

**How to fix**:
"Before I design, let me clarify requirements. What features are in scope? What's the expected scale? What are the latency requirements?"

**Better example**:
> Interviewer: "Design a URL shortener."
> 
> Candidate: "Before I start designing, let me clarify a few things:
> - What's the expected scale? How many URLs per day?
> - What features are in scope? Just shortening, or also analytics, custom URLs, expiration?
> - What's the read/write ratio? How many redirects vs. new URLs?
> - Any latency requirements? What's acceptable for redirects?
> - Should shortened URLs expire, or are they permanent?"
> 
> This shows senior-level thinking and ensures you design the right system.

**Time investment**: 5-7 minutes on requirements saves redesign later.

---

### Mistake 2: Not Clarifying Requirements

**What it looks like**:
Candidate assumes features, scale, and constraints without asking.

**Specific examples**:
- **Example 1**: Designing a social media feed without asking if it's chronological, algorithmic, or both. Building the wrong type wastes time.
- **Example 2**: Assuming 1 million users when the interviewer meant 1 billion. Designing a single-server solution for a billion-user system.
- **Example 3**: Building real-time features when the requirement is eventual consistency. Over-engineering with WebSockets when polling would suffice.
- **Example 4**: Including analytics, recommendations, and notifications when the interviewer only asked for core functionality. Scope creep wastes time.
- **Example 5**: Not asking about geographic distribution. Designing for a single region when global distribution is required.

**Why it's bad**:
- May design for 1M users when interviewer meant 1B
- May include features that are out of scope
- May miss critical non-functional requirements
- Wastes time on wrong design

**How to fix**:
Use a checklist:
- Functional: What features?
- Scale: How many users/requests?
- Latency: What response times?
- Consistency: Strong or eventual?
- Availability: What uptime?

**Better example**:
> Interviewer: "Design a video streaming service."
> 
> Candidate: "Let me clarify requirements:
> - **Features**: Live streaming, on-demand, or both? Do we need recommendations, comments, subscriptions?
> - **Scale**: How many concurrent viewers? Peak vs. average?
> - **Latency**: For live streaming, what's acceptable? Sub-second? A few seconds?
> - **Quality**: Multiple quality levels? Adaptive bitrate?
> - **Geographic**: Global or specific regions?
> - **Storage**: How long do videos stay? Forever or expire after X days?"
> 
> This ensures you build the right system from the start.

---

### Mistake 3: Ignoring Non-Functional Requirements

**What it looks like**:
Candidate designs features but never discusses scale, latency, or availability.

**Specific examples**:
- **Example 1**: Designing a notification system without asking about latency requirements. Building a batch system when real-time (<100ms) is needed.
- **Example 2**: Not asking about availability. Designing a single-server system when 99.99% uptime (4 minutes downtime/year) is required.
- **Example 3**: Ignoring consistency requirements. Using eventual consistency when strong consistency is required (e.g., financial transactions).
- **Example 4**: Not considering durability. Using in-memory storage when zero data loss is required.
- **Example 5**: Missing scale requirements. Designing for 1000 QPS when the system needs 1 million QPS.

**Why it's bad**:
- Design may not work at required scale
- Shows lack of production experience
- Misses opportunity to demonstrate depth
- Senior roles require NFR awareness

**How to fix**:
Always ask about and address:
- Scale (users, QPS, data volume)
- Latency (p50, p99 targets)
- Availability (99.9%? 99.99%?)
- Consistency (strong vs eventual)
- Durability (data loss tolerance)

**Better example**:
> Candidate: "For this notification system, I need to understand:
> - **Latency**: What's acceptable? Real-time (<100ms) or is 1-2 seconds okay?
> - **Availability**: What uptime is required? 99.9% or 99.99%?
> - **Scale**: How many notifications per second? Peak vs. average?
> - **Consistency**: If a user sends a message, does the recipient need to see it immediately, or is eventual delivery okay?
> - **Durability**: Can we lose notifications if a server crashes, or must they be guaranteed?"
> 
> This shows production thinking and ensures the design meets requirements.

---

### Mistake 4: Over-Engineering

**What it looks like**:
Designing microservices, Kubernetes, and global distribution for a system that serves 1000 users.

**Specific examples**:
- **Example 1**: Designing a microservices architecture with service mesh, API gateway, and distributed tracing for a system with 10,000 users. A monolith would be simpler and cheaper.
- **Example 2**: Proposing Kafka for a system that processes 100 messages per second. A simple message queue would suffice.
- **Example 3**: Designing a multi-region, globally distributed system for a local business application. Single region is fine.
- **Example 4**: Using Redis Cluster with 6 nodes for caching 1GB of data. A single Redis instance is sufficient.
- **Example 5**: Proposing Kubernetes for deploying 3 services. Docker Compose or simple VMs would be simpler.

**Why it's bad**:
- Shows poor judgment
- Adds unnecessary complexity
- Increases cost and operational burden
- May indicate inability to prioritize

**How to fix**:
Match complexity to requirements:
- 10K users: Single server might be fine
- 100K users: Load balancer + few servers
- 1M users: Start thinking about scaling patterns
- 10M+ users: Now microservices make sense

Ask yourself: "Would I actually build this for the given scale?"

**Better example**:
> Interviewer: "Design a system for 50,000 users."
> 
> Candidate: "For 50K users, I'd start simple:
> - Single application server (can handle ~10K concurrent users)
> - Load balancer in front (for redundancy and future scaling)
> - Single database with read replicas (can handle the load)
> - Simple caching layer (Redis)
> 
> I wouldn't use microservices yet - that adds complexity without clear benefit at this scale. If we grow to 5M users, then we'd consider microservices. Does this approach make sense?"
> 
> This shows good judgment and cost awareness.

---

### Mistake 5: Under-Engineering

**What it looks like**:
"We'll just use a single PostgreSQL database" for a system expecting 1 million QPS.

**Specific examples**:
- **Example 1**: Proposing a single MySQL database for a system with 1 million QPS. A single database can handle ~10K QPS, so this won't work.
- **Example 2**: Using a single Redis instance for caching 10TB of data. Redis is in-memory and limited by RAM.
- **Example 3**: Designing a single-server solution for a global system with users in 50 countries. Latency would be terrible.
- **Example 4**: Not considering replication or backups for a critical financial system. Single point of failure.
- **Example 5**: Using synchronous processing for a system that needs to handle 100K requests/second. Would be too slow.

**Why it's bad**:
- Design won't work at required scale
- Shows lack of scaling knowledge
- Interviewer loses confidence in your abilities
- May indicate lack of production experience

**How to fix**:
Do capacity estimation early:
- Calculate QPS
- Estimate storage needs
- Identify bottlenecks
- Scale appropriately

**Better example**:
> Interviewer: "Design a system that handles 1 million QPS."
> 
> Candidate: "Let me do some quick math:
> - 1M QPS = 86.4 billion requests per day
> - A single database can handle ~10K QPS, so we'd need 100+ database instances
> - But that's inefficient. Instead, I'd:
>   - Use read replicas (most reads can go to replicas)
>   - Add caching (Redis) to reduce database load by 80-90%
>   - Shard the database (partition by user_id)
>   - Use CDN for static content
> 
> This would handle the scale. Should I detail the sharding strategy?"
> 
> This shows you understand scale and know how to handle it.

---

### Mistake 6: Not Discussing Trade-offs

**What it looks like**:
"I'll use Redis for caching." (No explanation of why, or what alternatives were considered.)

**Specific examples**:
- **Example 1**: "I'll use MySQL" without mentioning PostgreSQL, NoSQL options, or why MySQL fits this use case.
- **Example 2**: "We'll use microservices" without discussing monolith vs. microservices trade-offs.
- **Example 3**: "I'll use Kafka" without mentioning RabbitMQ, SQS, or when you'd use each.
- **Example 4**: "We'll use eventual consistency" without explaining when strong consistency is needed and the trade-offs.
- **Example 5**: "I'll use CDN" without discussing cost, cache invalidation challenges, or when it's not needed.

**Why it's bad**:
- Misses chance to show depth
- Interviewer can't evaluate your reasoning
- Suggests you're following patterns blindly
- Senior roles require trade-off analysis

**How to fix**:
For every major decision:
1. State what you chose
2. Mention alternatives considered
3. Explain why you chose this option
4. Acknowledge the trade-offs

**Better examples**:

> **Example 1 - Database choice**:
> "I'll use PostgreSQL for the primary database. I considered MySQL, but PostgreSQL has better support for JSON queries which we'll need. I also considered NoSQL like MongoDB, but we need ACID transactions for financial data. The trade-off is PostgreSQL is slightly more complex to operate than MySQL, but the features are worth it."

> **Example 2 - Caching**:
> "I'll use Redis for caching. I considered Memcached, which is simpler, but Redis supports rich data structures (sets, sorted sets) that we'll need for leaderboards. I also considered in-memory caching in the application, but we need shared cache across multiple servers. The trade-off is Redis requires more memory management, but it's worth it for the features."

> **Example 3 - Message Queue**:
> "I'll use Kafka for event streaming. I considered RabbitMQ, which is simpler, but Kafka's partitioning and ordering guarantees are important for our use case. I also considered SQS, but we need exactly-once delivery which Kafka provides. The trade-off is Kafka is more complex to operate, but it fits our requirements better."

---

### Mistake 7: Poor Time Management

**What it looks like**:
Spending 20 minutes on database schema, then rushing through everything else.

**Specific examples**:
- **Example 1**: Spending 25 minutes designing the perfect database schema, then having only 5 minutes for the rest of the system. Interviewer never sees your scaling or reliability thinking.
- **Example 2**: Getting stuck on one component (e.g., how to generate unique IDs) for 15 minutes, leaving no time for the overall architecture.
- **Example 3**: Drawing a beautiful diagram with every detail, spending 30 minutes, then rushing through the deep dive.
- **Example 4**: Going too deep on caching strategy (20 minutes) and not covering failure scenarios, monitoring, or scaling.
- **Example 5**: Spending the entire interview on high-level design without any deep dive, missing the chance to show technical depth.

**Why it's bad**:
- Incomplete design
- Missed opportunity to show breadth
- Interviewer can't evaluate all areas
- Suggests poor prioritization skills

**How to fix**:
Use checkpoints:
- 8 min: Requirements done
- 12 min: Estimation done
- 28 min: High-level design done
- 40 min: Deep dive done
- 45 min: Wrap-up done

Glance at clock at each checkpoint. Adjust if behind.

**Better example**:
> Candidate (at 10 minutes): "I've clarified requirements and done capacity estimation. I have 35 minutes left. Should I focus on the high-level architecture first, or is there a specific area you'd like me to prioritize?"
> 
> Candidate (at 25 minutes): "I've covered the high-level design. I have 20 minutes left. I'd like to deep dive into the database sharding strategy and caching layer. Does that work, or would you prefer I cover something else?"
> 
> This shows time awareness and collaboration.

---

### Mistake 8: Not Drawing Diagrams

**What it looks like**:
Explaining the entire design verbally without using the whiteboard.

**Specific examples**:
- **Example 1**: Describing a microservices architecture with 10 services verbally. Interviewer can't keep track of all the services and their relationships.
- **Example 2**: Explaining data flow through multiple systems without drawing it. "So the user request goes to the API gateway, then to the auth service, then to the main service, then to the database, then back..." - too hard to follow.
- **Example 3**: Describing a distributed system with multiple regions without a diagram. Interviewer can't visualize the architecture.
- **Example 4**: Explaining caching layers, CDN, load balancers, and databases all verbally. The complexity is lost without visualization.
- **Example 5**: Describing a message queue architecture with producers, consumers, and topics without drawing it. Relationships are unclear.

**Why it's bad**:
- Hard to follow complex systems verbally
- Interviewer can't see the big picture
- Misses chance to demonstrate communication skills
- May forget components without visual reference

**How to fix**:
- Always draw a diagram
- Start simple, add detail
- Label all components
- Show data flow with arrows
- Update diagram as design evolves

**Better example**:
> Candidate: "Let me draw the high-level architecture. [Draws diagram]
> 
> We have:
> - Users connecting through a CDN
> - Load balancer distributing traffic
> - API servers (stateless, can scale horizontally)
> - Cache layer (Redis)
> - Database with read replicas
> - Message queue for async processing
> 
> [Points to diagram] The flow is: User → CDN → Load Balancer → API Server → Cache/Database. Does this make sense so far?"
> 
> Visual communication is much clearer than verbal only.

---

### Mistake 9: Not Asking Questions

**What it looks like**:
Candidate never checks in with interviewer, never asks for clarification.

**Specific examples**:
- **Example 1**: Candidate designs for 45 minutes straight without pausing. Interviewer wanted to redirect 20 minutes ago but couldn't interrupt.
- **Example 2**: Candidate goes deep on database design when interviewer wanted to see scaling strategy. Never asked what to focus on.
- **Example 3**: Candidate designs a solution the interviewer knows won't work, but never checks in. Interviewer has to stop them and redirect.
- **Example 4**: Candidate makes assumptions about requirements and never verifies. Designs the wrong system.
- **Example 5**: Interviewer gives hints ("What about caching?") but candidate doesn't pick up on them. Misses opportunities to show knowledge.

**Why it's bad**:
- May be going in wrong direction
- Misses hints from interviewer
- Shows poor collaboration skills
- Interview feels like a monologue

**How to fix**:
Check in regularly:
- "Does this make sense so far?"
- "Should I go deeper here or move on?"
- "Is there a specific area you'd like me to focus on?"
- "Am I on the right track?"

**Better example**:
> Candidate (after 10 minutes): "I've clarified requirements and done capacity estimation. Before I continue with the design, does this approach make sense, or would you like me to adjust anything?"
> 
> Candidate (after 20 minutes): "I've covered the high-level architecture. Should I go deeper into any specific component, or continue with the overall design?"
> 
> Candidate (when stuck): "I'm considering two approaches here. Approach A is simpler but less scalable. Approach B is more complex but handles scale better. Which direction would you like me to go?"
> 
> This shows collaboration and adaptability.

---

### Mistake 10: Getting Defensive When Challenged

**What it looks like**:
Interviewer: "Why not use NoSQL here?"
Candidate: "SQL is fine. NoSQL has its own problems."

**Specific examples**:
- **Example 1**: Interviewer: "What about using a message queue here?" Candidate: "I don't think we need it. Direct calls are fine." (Defensive, doesn't consider the suggestion)
- **Example 2**: Interviewer: "Have you considered eventual consistency?" Candidate: "No, we need strong consistency. Eventual consistency is too complex." (Dismissive)
- **Example 3**: Interviewer: "What about caching?" Candidate: "I already covered that. We're using Redis." (Defensive, doesn't engage)
- **Example 4**: Interviewer: "This might not scale. What if you have 10x the traffic?" Candidate: "The design I showed will scale fine. We can just add more servers." (Doesn't address the concern)
- **Example 5**: Interviewer: "I'm concerned about the single point of failure." Candidate: "That's not a problem. The database is reliable." (Dismissive of valid concern)

**Why it's bad**:
- Misses chance to show flexibility
- Interviewer may have been hinting at a better approach
- Shows inability to receive feedback
- Red flag for collaboration

**How to fix**:
Treat challenges as opportunities:

**Better examples**:

> **Example 1**:
> Interviewer: "Why not use NoSQL here?"
> 
> Candidate: "Good question. Let me think about that. NoSQL would give us better horizontal scaling and schema flexibility. The trade-off is we'd lose ACID transactions and complex queries. Given our requirements for financial transactions, I still lean toward SQL for strong consistency. But if we didn't need ACID, NoSQL could work well. What's your thinking - are you seeing a specific benefit I'm missing?"

> **Example 2**:
> Interviewer: "This might not scale. What if you have 10x the traffic?"
> 
> Candidate: "That's a great point. Let me think through that. At 10x scale, the database would definitely be a bottleneck. I'd need to:
> - Add more read replicas
> - Implement database sharding
> - Increase cache hit rates
> - Consider moving some operations to async processing
> 
> You're right - I should have addressed that. Should I detail the sharding strategy?"

> **Example 3**:
> Interviewer: "Have you considered eventual consistency?"
> 
> Candidate: "I considered it, but chose strong consistency because [reason]. However, you raise a good point - if we can accept eventual consistency for some operations, we could significantly improve performance. For example, user profiles could be eventually consistent, while financial transactions need strong consistency. Should I redesign with that in mind?"

This shows you can receive feedback, think flexibly, and collaborate.

---

### Mistake 11: Giving Vague Answers

**What it looks like**:
"We'd use caching to make it faster."
"We'd scale horizontally."
"We'd use a message queue."

**Specific examples**:
- **Example 1**: "We'll use a database" instead of "We'll use PostgreSQL with read replicas, sharded by user_id, with connection pooling."
- **Example 2**: "We'll add caching" instead of "We'll use Redis as a cache-aside cache with 1-hour TTL, expecting 95% hit rate based on access patterns."
- **Example 3**: "We'll scale it" instead of "We'll scale horizontally by adding stateless API servers behind a load balancer, with auto-scaling triggered at 70% CPU."
- **Example 4**: "We'll use a queue" instead of "We'll use Kafka for async processing, partitioned by user_id to maintain ordering per user."
- **Example 5**: "We'll use CDN" instead of "We'll use CloudFront to cache static assets at edge locations, with cache invalidation via API when content updates."

**Why it's bad**:
- Doesn't demonstrate understanding
- Interviewer can't evaluate depth
- Sounds like memorized buzzwords
- Doesn't differentiate from other candidates

**How to fix**:
Be specific:

**Better examples**:

> **Vague**: "We'll use caching."
> **Specific**: "We'll use Redis as a cache-aside cache with 1-hour TTL. Based on access patterns, I'd expect 95% hit rate, reducing database load by 95%. We'll invalidate cache on writes using event-based invalidation."

> **Vague**: "We'll scale horizontally."
> **Specific**: "We'll scale horizontally by adding stateless API servers behind a load balancer. We'll use auto-scaling that triggers at 70% CPU, scaling up to 50 instances during peak and down to 5 during off-peak. Each server can handle ~1000 requests/second."

> **Vague**: "We'll use a message queue."
> **Specific**: "We'll use Kafka for async processing of notifications. We'll partition by user_id to maintain ordering per user. We'll have 10 partitions to allow parallel processing, with consumer groups for different notification types (email, push, SMS)."

> **Vague**: "We'll use a database."
> **Specific**: "We'll use PostgreSQL as the primary database with 3 read replicas. We'll shard by user_id across 10 shards. Each shard will have its own replica for read scaling. We'll use connection pooling (max 100 connections per server) to manage database connections efficiently."

Specificity shows real understanding, not just buzzword knowledge.

---

### Mistake 12: Not Handling Failure Scenarios

**What it looks like**:
Design only covers the happy path. No discussion of what happens when things fail.

**Specific examples**:
- **Example 1**: Designing a system with a single database and never mentioning what happens if it fails. No backup, no replication, no failover plan.
- **Example 2**: Using a cache but not discussing cache misses, cache failures, or what happens when cache is unavailable.
- **Example 3**: Designing a distributed system but not mentioning network partitions, service failures, or how to handle partial failures.
- **Example 4**: Using a message queue but not discussing what happens if the queue is down, messages are lost, or consumers fail.
- **Example 5**: Designing a CDN but not discussing what happens if CDN fails, cache is stale, or origin server is down.

**Why it's bad**:
- Production systems fail
- Shows lack of operational experience
- Misses chance to demonstrate depth
- Senior roles require reliability thinking

**How to fix**:
For each major component, consider:
- What happens if it fails?
- How do we detect failure?
- How do we recover?
- What's the blast radius?

**Better examples**:

> **Database failure**:
> "If the primary database fails, we have automatic failover to a replica. Detection is via health checks every 10 seconds. Failover takes 1-2 minutes. During failover, writes fail but reads continue from other replicas. We also have a standby replica in a different region for disaster recovery. The blast radius is limited to writes during the 1-2 minute failover window."

> **Cache failure**:
> "If Redis fails, the system degrades gracefully. We detect failure via health checks. On cache failure, requests go directly to the database. This increases latency from 1ms to 50ms, but the system remains functional. We have Redis replication, so if the primary fails, we failover to the replica within 30 seconds. The blast radius is increased latency, not service unavailability."

> **Service failure**:
> "If an API server fails, the load balancer detects it via health checks and stops routing traffic to it. Other servers continue handling requests. We have auto-scaling, so if multiple servers fail, we automatically spin up replacements. The blast radius is reduced capacity, but service remains available. We also have circuit breakers to prevent cascading failures."

> **Message queue failure**:
> "If Kafka fails, we have multiple brokers in the cluster. If one broker fails, partitions are replicated to other brokers. If the entire cluster fails, we have a disaster recovery setup with async replication to a secondary cluster. During failure, we can't process new messages, but we don't lose messages due to replication. The blast radius is delayed processing, not data loss."

This shows production thinking and operational maturity.

---

### Mistake 13: Memorizing Without Understanding

**What it looks like**:
Can recite that "Kafka is used for event streaming" but can't explain when to use it vs RabbitMQ, or how it handles ordering.

**Specific examples**:
- **Example 1**: Says "We'll use microservices" but can't explain when microservices make sense vs. a monolith, or the operational overhead.
- **Example 2**: Says "We'll use Redis for caching" but can't explain cache eviction policies, memory management, or when Redis isn't appropriate.
- **Example 3**: Says "We'll use NoSQL" but can't explain CAP theorem, consistency models, or when SQL is better.
- **Example 4**: Says "We'll use Kubernetes" but can't explain what problems it solves, when it's overkill, or alternatives.
- **Example 5**: Says "We'll use CDN" but can't explain cache invalidation, when CDN doesn't help, or cost considerations.

**Why it's bad**:
- Falls apart under follow-up questions
- Interviewer detects surface-level knowledge
- Can't adapt to novel problems
- Doesn't demonstrate real understanding

**How to fix**:
For each technology, understand:
- What problem does it solve?
- How does it work internally?
- What are the trade-offs?
- When would you NOT use it?
- What are alternatives?

**Better examples**:

> **Kafka understanding**:
> "I'll use Kafka for event streaming. Kafka solves the problem of high-throughput, ordered event processing. It works by partitioning topics, where each partition maintains order. Messages are persisted to disk and replicated across brokers. 
> 
> Trade-offs: Kafka is great for high throughput and ordering, but it's more complex than simpler queues like RabbitMQ. RabbitMQ is better for lower throughput, simpler routing, and when you don't need ordering guarantees.
> 
> I wouldn't use Kafka if I just need simple task queues or if throughput is low (<1000 messages/sec). For that, I'd use RabbitMQ or SQS.
> 
> Alternatives include RabbitMQ (simpler, lower throughput), AWS Kinesis (managed, similar features), or Google Pub/Sub (managed, global)."

> **Microservices understanding**:
> "I'll use microservices here. Microservices solve the problem of scaling different parts of a system independently and allowing teams to work independently. They work by splitting a monolith into smaller, independently deployable services.
> 
> Trade-offs: Microservices provide better scalability and team autonomy, but they add complexity - service discovery, distributed tracing, network latency, eventual consistency challenges.
> 
> I wouldn't use microservices for a small team (<10 engineers) or low-traffic system (<100K users). For that, a monolith is simpler and faster to develop.
> 
> Alternatives include monolith (simpler, faster development), modular monolith (middle ground), or serverless (for event-driven architectures)."

Deep understanding shows you can make informed decisions, not just follow patterns.

---

### Mistake 14: Ignoring the Interviewer

**What it looks like**:
Candidate talks continuously without pausing, ignores interviewer's attempts to interject.

**Specific examples**:
- **Example 1**: Candidate talks for 10 minutes straight. Interviewer tries to interject with "What about..." but candidate keeps talking.
- **Example 2**: Interviewer asks a question, but candidate says "Let me finish this first" and continues for 5 more minutes.
- **Example 3**: Interviewer gives a hint ("Have you considered X?") but candidate doesn't acknowledge it and continues with their original approach.
- **Example 4**: Interviewer looks confused or starts to say something, but candidate doesn't pause to check if they're following.
- **Example 5**: Interviewer redirects ("Let's focus on Y instead"), but candidate continues with X.

**Why it's bad**:
- Misses hints and guidance
- Shows poor communication skills
- Interview feels one-sided
- May go down wrong path

**How to fix**:
- Pause regularly
- Watch for interviewer signals
- Invite questions
- Adapt based on feedback

**Better example**:
> Candidate: "So the architecture has three main components... [pauses, looks at interviewer] Does this make sense so far, or should I clarify anything?"
> 
> Interviewer: "What about caching?"
> 
> Candidate: "Good point! I was about to cover that. Let me add caching to the design. [Draws cache layer] We'll use Redis here for..."
> 
> Candidate: [After explaining a complex part] "I want to make sure I'm explaining this clearly. Are you following, or should I break it down differently?"
> 
> This shows awareness, collaboration, and communication skills.

---

### Mistake 15: Not Summarizing

**What it looks like**:
Design ends abruptly without summary or wrap-up.

**Specific examples**:
- **Example 1**: Candidate finishes designing the last component and says "That's it" without summarizing the overall system.
- **Example 2**: Time runs out and candidate stops mid-sentence. No summary of what was covered.
- **Example 3**: Candidate completes the design but doesn't recap key decisions, trade-offs, or next steps.
- **Example 4**: Interviewer asks "Anything else?" and candidate says "No" without summarizing what was designed.
- **Example 5**: Candidate covers many components but doesn't tie them together in a final summary.

**Why it's bad**:
- Misses chance to reinforce key points
- Interviewer may forget earlier parts
- No closure to the discussion
- Seems incomplete

**How to fix**:
Always end with a summary:

**Better examples**:

> **Good summary**:
> "To summarize what we've designed:
> - A URL shortener that handles 100M URLs/day
> - Key components: API servers (stateless, horizontally scalable), Redis cache (95% hit rate), PostgreSQL database (sharded by hash of short URL), and a base62 encoding service
> - Main trade-off: We chose SQL over NoSQL for ACID guarantees, accepting that sharding adds complexity
> - For monitoring, I'd track: QPS, cache hit rate, database query latency, and error rates
> - Future improvements: Add analytics, custom URLs, URL expiration, and geographic distribution for lower latency
> 
> Does this cover what you were looking for, or would you like me to dive deeper into any area?"

> **Another good summary**:
> "Let me recap the design:
> - System: Real-time chat supporting 10M concurrent users
> - Architecture: WebSocket servers, message queue (Kafka), database (Cassandra for message history), and Redis for presence
> - Key decisions: 
>   - WebSockets for real-time (vs. polling) - needed for <100ms latency
>   - Kafka for message ordering and durability
>   - Cassandra for scalable message storage
> - Trade-offs: Added complexity for scale and reliability
> - Monitoring: Connection count, message latency, queue depth, error rates
> - Next steps: Add message search, file sharing, and end-to-end encryption
> 
> Is there anything you'd like me to elaborate on?"

Summarizing shows you can synthesize information and communicate clearly - key skills for senior engineers.

---

## 3️⃣ Mistake Severity Guide

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MISTAKE SEVERITY GUIDE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  CRITICAL (Can fail you alone)                                      │
│  ─────────────────────────────                                      │
│  □ Jumping to solution without requirements                         │
│  □ Design that doesn't work at required scale                       │
│  □ Getting defensive when challenged                                │
│  □ Fundamental misunderstanding of core concepts                    │
│                                                                      │
│  MAJOR (Significantly hurts your chances)                           │
│  ─────────────────────────────────────────                          │
│  □ Poor time management (incomplete design)                         │
│  □ No trade-off discussion                                          │
│  □ Ignoring non-functional requirements                             │
│  □ Over/under-engineering                                           │
│                                                                      │
│  MODERATE (Noticeable but recoverable)                              │
│  ─────────────────────────────────────                              │
│  □ Not drawing diagrams                                             │
│  □ Vague answers                                                    │
│  □ Not handling failure scenarios                                   │
│  □ Not checking in with interviewer                                 │
│                                                                      │
│  MINOR (Small negative signal)                                      │
│  ─────────────────────────────                                      │
│  □ Not summarizing at end                                           │
│  □ Minor time management issues                                     │
│  □ Occasional vague answer                                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Self-Assessment Checklist

Use this checklist after mock interviews:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    POST-INTERVIEW SELF-ASSESSMENT                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  REQUIREMENTS PHASE                                                 │
│  □ Did I clarify functional requirements?                           │
│  □ Did I ask about scale?                                           │
│  □ Did I ask about latency/consistency?                             │
│  □ Did I identify what's out of scope?                              │
│                                                                      │
│  ESTIMATION PHASE                                                   │
│  □ Did I estimate QPS?                                              │
│  □ Did I estimate storage?                                          │
│  □ Did I identify bottlenecks?                                      │
│                                                                      │
│  HIGH-LEVEL DESIGN                                                  │
│  □ Did I draw a clear diagram?                                      │
│  □ Did I explain each component?                                    │
│  □ Did I show data flow?                                            │
│  □ Did I cover all major components?                                │
│                                                                      │
│  DEEP DIVE                                                          │
│  □ Did I go deep on 1-2 components?                                 │
│  □ Did I discuss algorithms/data structures?                        │
│  □ Did I handle edge cases?                                         │
│  □ Did I discuss failure scenarios?                                 │
│                                                                      │
│  TRADE-OFFS                                                         │
│  □ Did I discuss trade-offs for major decisions?                    │
│  □ Did I mention alternatives?                                      │
│  □ Did I justify my choices?                                        │
│                                                                      │
│  COMMUNICATION                                                      │
│  □ Did I think out loud?                                            │
│  □ Did I check in with the interviewer?                             │
│  □ Did I manage time well?                                          │
│  □ Did I summarize at the end?                                      │
│                                                                      │
│  ATTITUDE                                                           │
│  □ Did I handle pushback gracefully?                                │
│  □ Did I stay calm under pressure?                                  │
│  □ Did I show enthusiasm?                                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5️⃣ One Clean Mental Summary

The 15 most common mistakes in system design interviews are predictable and avoidable. The critical mistakes that can fail you alone: jumping to solution without requirements, designs that don't scale, getting defensive, and fundamental misunderstandings.

The major mistakes that significantly hurt your chances: poor time management, no trade-off discussion, ignoring NFRs, and over/under-engineering.

Avoid these mistakes by: always clarifying requirements first, doing capacity estimation, drawing diagrams, discussing trade-offs for every major decision, checking in with the interviewer regularly, handling challenges gracefully, and summarizing at the end.

Use the self-assessment checklist after every mock interview to identify which mistakes you're making. Deliberately practice avoiding your most common mistakes until good habits become automatic.

