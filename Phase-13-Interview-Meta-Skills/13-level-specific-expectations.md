# Level-Specific Expectations

## 0️⃣ Prerequisites

Before diving into level-specific expectations, you should understand:

- **Problem Approach Framework**: How to structure your interview (covered in Topic 1)
- **System Design Fundamentals**: Core components and patterns (covered in Phases 1-9)
- **Trade-off Analysis**: How to evaluate and justify decisions (covered in Topic 9)

Quick refresher: Different engineering levels have different expectations in system design interviews. Understanding what's expected at your target level helps you calibrate your responses appropriately. Over-engineering for an L4 role or under-delivering for an L6 role both hurt your chances.

---

## 1️⃣ Why Level Expectations Matter

Interviewers calibrate their evaluation based on the level you're interviewing for:

- **L4 (Entry/New Grad)**: Can you design a working system with guidance?
- **L5 (Mid-level)**: Can you design end-to-end independently?
- **L6 (Senior)**: Can you propose multiple solutions with complex trade-offs?
- **L7 (Staff)**: Can you design systems of systems with organizational impact?

Knowing these expectations helps you:
1. **Calibrate depth**: How deep to go on each topic
2. **Prioritize content**: What to emphasize vs skim
3. **Demonstrate appropriate skills**: What the interviewer is looking for
4. **Avoid over/under-engineering**: Match complexity to level

---

## 2️⃣ Level Breakdown

### L4 (Entry Level / New Grad)

**What interviewers expect**:
- Basic understanding of system components
- Can design a working solution with guidance
- Knows common trade-offs at a high level
- May need hints to cover all areas
- Demonstrates learning ability

**What's NOT expected**:
- Deep expertise in specific technologies
- Complex distributed systems knowledge
- Production experience with scale
- Handling all edge cases unprompted

```
┌─────────────────────────────────────────────────────────────────────┐
│                    L4 EXPECTATIONS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  REQUIREMENTS CLARIFICATION                                         │
│  □ Asks basic clarifying questions                                  │
│  □ Identifies core functional requirements                          │
│  □ May need prompting for non-functional requirements               │
│                                                                      │
│  HIGH-LEVEL DESIGN                                                  │
│  □ Draws a reasonable architecture                                  │
│  □ Identifies main components (API, DB, cache)                      │
│  □ Shows basic data flow                                            │
│  □ May miss some components (interviewer guides)                    │
│                                                                      │
│  DEEP DIVE                                                          │
│  □ Can explain one component in detail                              │
│  □ Understands basic database concepts                              │
│  □ Knows what caching is and why it helps                           │
│  □ May struggle with distributed systems concepts                   │
│                                                                      │
│  TRADE-OFFS                                                         │
│  □ Understands SQL vs NoSQL at high level                           │
│  □ Knows caching has invalidation challenges                        │
│  □ May not articulate trade-offs unprompted                         │
│                                                                      │
│  COMMUNICATION                                                      │
│  □ Explains thinking process                                        │
│  □ Asks for help when stuck                                         │
│  □ Receptive to hints and guidance                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Example L4 response snippet**:

"For the URL shortener, I'll use a web server to handle requests, a database to store the URL mappings, and maybe a cache to speed up lookups.

When a user submits a long URL, we generate a short code, store the mapping in the database, and return the short URL. When someone accesses the short URL, we look up the mapping and redirect them.

For the database, I'd use something like PostgreSQL because I'm familiar with it. For the short code, maybe we could use a hash of the URL?

*[Interviewer: What about collisions with the hash?]*

Oh right, we'd need to handle that. Maybe we could check if the short code already exists and regenerate if it does. Or use an auto-incrementing ID and convert it to base62."

**What makes this L4-appropriate**:
- Basic working design
- Reasonable component choices
- Acknowledges gaps when prompted
- Shows learning/adapting

---

### L5 (Mid-Level / Senior SWE)

**What interviewers expect**:
- Drives the interview independently
- Covers all major areas without prompting
- Understands distributed systems basics
- Can do capacity estimation
- Discusses trade-offs proactively
- Has production experience to draw from

**What's NOT expected**:
- Expertise in every technology
- Designing for extreme scale (billions of users)
- Organizational/team considerations
- Multiple complete alternative designs

```
┌─────────────────────────────────────────────────────────────────────┐
│                    L5 EXPECTATIONS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  REQUIREMENTS CLARIFICATION                                         │
│  □ Systematically covers functional requirements                    │
│  □ Proactively asks about scale, latency, consistency               │
│  □ Identifies out-of-scope items                                    │
│  □ Does capacity estimation                                         │
│                                                                      │
│  HIGH-LEVEL DESIGN                                                  │
│  □ Complete architecture covering all components                    │
│  □ Explains purpose of each component                               │
│  □ Shows data flow for main use cases                               │
│  □ Considers failure scenarios                                      │
│                                                                      │
│  DEEP DIVE                                                          │
│  □ Can deep dive on any component                                   │
│  □ Understands database scaling (replicas, sharding)                │
│  □ Knows caching patterns and invalidation strategies               │
│  □ Understands async processing with queues                         │
│  □ Can discuss consistency models                                   │
│                                                                      │
│  TRADE-OFFS                                                         │
│  □ Proactively discusses trade-offs                                 │
│  □ Compares alternatives (SQL vs NoSQL, etc.)                       │
│  □ Justifies choices based on requirements                          │
│  □ Acknowledges downsides of chosen approach                        │
│                                                                      │
│  COMMUNICATION                                                      │
│  □ Drives the conversation forward                                  │
│  □ Checks in with interviewer                                       │
│  □ Manages time effectively                                         │
│  □ Clear and structured explanations                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Example L5 response snippet**:

"Before I design, let me clarify requirements. For functional requirements, we need to create short URLs and redirect to the original. Are custom short URLs in scope? What about analytics?

For non-functional requirements, what's the expected scale? Let me estimate: if we have 100 million URLs created per month and a 10:1 read/write ratio, that's about 1 billion redirects per month, or roughly 400 QPS average, 1200 QPS peak.

For storage, 100 million URLs per month at 500 bytes each is 50 GB per month. Over 5 years, that's 3 TB, which is manageable.

Given this scale, here's my architecture:

*[Draws complete diagram with load balancer, API servers, cache, database, and CDN]*

For the database, I'll use PostgreSQL with read replicas. At 1200 QPS, a single primary can handle writes, and replicas handle reads. If we grow 10x, we'd need to consider sharding.

For caching, I'll use Redis with cache-aside pattern. Since URLs are immutable once created, we can cache aggressively with long TTL. I'd expect 95%+ cache hit rate, reducing database load significantly.

For ID generation, I'll use a Snowflake-like approach rather than auto-increment. This allows us to scale writes across multiple servers without coordination.

The main trade-off is complexity vs scalability. For our current scale, this might be over-engineered. But it gives us room to grow without major redesign."

**What makes this L5-appropriate**:
- Drives independently
- Covers all major areas
- Does capacity estimation
- Proactively discusses trade-offs
- Mentions scaling path

---

### L6 (Senior / Staff)

**What interviewers expect**:
- Multiple solution approaches with trade-offs
- Deep expertise in chosen technologies
- Considers operational concerns (monitoring, deployment)
- Handles complex distributed systems scenarios
- Thinks about evolution and future requirements
- May discuss team/organizational considerations

**What's NOT expected**:
- Knowing every technology in depth
- Solving problems no one has solved before
- Perfect answers to every question

```
┌─────────────────────────────────────────────────────────────────────┐
│                    L6 EXPECTATIONS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  REQUIREMENTS CLARIFICATION                                         │
│  □ Uncovers hidden requirements through probing                     │
│  □ Identifies potential conflicts in requirements                   │
│  □ Suggests trade-offs between requirements                         │
│  □ Considers future evolution                                       │
│                                                                      │
│  HIGH-LEVEL DESIGN                                                  │
│  □ Presents multiple approaches before choosing                     │
│  □ Architecture handles edge cases                                  │
│  □ Considers operational concerns (deployment, monitoring)          │
│  □ Designs for evolution and extensibility                          │
│                                                                      │
│  DEEP DIVE                                                          │
│  □ Expert-level knowledge in multiple areas                         │
│  □ Handles complex distributed systems scenarios                    │
│  □ Discusses failure modes and recovery                             │
│  □ Considers security implications                                  │
│  □ Knows production gotchas                                         │
│                                                                      │
│  TRADE-OFFS                                                         │
│  □ Articulates nuanced trade-offs                                   │
│  □ Quantifies trade-offs where possible                             │
│  □ Considers cost, complexity, and team factors                     │
│  □ Explains when to choose each alternative                         │
│                                                                      │
│  COMMUNICATION                                                      │
│  □ Could mentor others through the problem                          │
│  □ Adapts depth based on interviewer interest                       │
│  □ Connects technical decisions to business impact                  │
│  □ Demonstrates leadership in problem-solving                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Example L6 response snippet**:

"For the URL shortener, I see three main architectural approaches:

**Approach 1: Simple centralized**
- Single PostgreSQL database
- Works for up to ~10K QPS
- Pros: Simple, easy to operate
- Cons: Single point of failure, scaling limits

**Approach 2: Distributed with consistent hashing**
- Shard URLs across multiple databases
- Use consistent hashing for routing
- Pros: Scales horizontally
- Cons: More complex, cross-shard queries difficult

**Approach 3: Event-sourced with CQRS**
- Write events to Kafka
- Separate read and write stores
- Pros: Audit trail, flexible reads
- Cons: Eventual consistency, more infrastructure

Given our requirements of 100M URLs/month and high availability, I'd recommend Approach 2 with some elements of Approach 1 for simplicity initially.

Let me walk through the design:

*[Detailed architecture with multiple components]*

For ID generation, I want to discuss a subtle issue. If we use timestamp-based IDs like Snowflake, URLs created around the same time will have similar short codes. This could be a privacy concern, as someone could enumerate recently created URLs. We have options:
1. Accept this risk (it's low)
2. Add randomness to the ID
3. Use a separate random component

I'd recommend option 2, adding 10 bits of randomness. This prevents enumeration while keeping IDs mostly ordered for database efficiency.

For monitoring, I'd track:
- Redirect latency (p50, p95, p99)
- Cache hit rate (target >95%)
- Database replication lag
- Error rates by type

For deployment, I'd use blue-green deployment with gradual traffic shift. This allows us to roll back quickly if issues arise.

One operational concern: what happens if we need to migrate the database? With billions of URLs, this is non-trivial. I'd recommend building migration tooling from day one, even if we don't need it immediately."

**What makes this L6-appropriate**:
- Multiple approaches compared
- Subtle technical considerations
- Operational concerns addressed
- Future evolution considered
- Could mentor others through this

---

### L7 (Staff+ / Principal)

**What interviewers expect**:
- System of systems thinking
- Organizational and team considerations
- Long-term technical strategy
- Cross-functional impact (product, business)
- Industry-level perspective
- Mentorship and leadership signals

```
┌─────────────────────────────────────────────────────────────────────┐
│                    L7 EXPECTATIONS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  SCOPE                                                              │
│  □ Thinks beyond single system to ecosystem                         │
│  □ Considers impact on other teams/systems                          │
│  □ Addresses organizational scaling                                 │
│  □ Long-term technical vision                                       │
│                                                                      │
│  TECHNICAL DEPTH                                                    │
│  □ Expert in multiple domains                                       │
│  □ Knows industry best practices and why they exist                 │
│  □ Can challenge conventional wisdom appropriately                  │
│  □ Understands infrastructure at scale                              │
│                                                                      │
│  LEADERSHIP                                                         │
│  □ Makes decisions that enable team autonomy                        │
│  □ Considers build vs buy strategically                             │
│  □ Thinks about technical debt and investment                       │
│  □ Balances short-term and long-term                                │
│                                                                      │
│  COMMUNICATION                                                      │
│  □ Can explain to executives and junior engineers                   │
│  □ Connects technical decisions to business outcomes                │
│  □ Influences without authority                                     │
│  □ Drives alignment across teams                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Example L7 response snippet**:

"Before diving into the URL shortener design, let me understand the broader context. Is this a standalone product or part of a larger platform? Who are the users, internal teams or external customers? What's the business model?

*[After clarification]*

Given this is a platform service that other teams will depend on, my design priorities shift:

1. **API stability**: Other teams will build on this. Breaking changes are expensive.
2. **Multi-tenancy**: Different teams may have different SLAs.
3. **Self-service**: Teams shouldn't need our involvement for basic operations.
4. **Observability**: Teams need visibility into their usage.

Let me outline the architecture:

*[System of systems diagram showing the URL shortener as a platform service]*

For team structure, I'd recommend:
- Core platform team (3-4 engineers) owns the service
- Clear API contracts with versioning
- SLOs defined per tier (99.9% for standard, 99.99% for premium)

For technical strategy:
- Phase 1: MVP with single-tenant design (3 months)
- Phase 2: Multi-tenancy and self-service (6 months)
- Phase 3: Advanced features (analytics, A/B testing) (ongoing)

The build vs buy question: We could use Bitly's enterprise API. Cost would be ~$X/year at our scale. Building gives us:
- Customization for our specific needs
- No vendor dependency
- Learning opportunity for the team
- Long-term cost savings at scale

I'd recommend building because [specific reasons tied to company context].

For cross-team impact:
- Marketing team needs custom branded domains
- Analytics team needs click-stream data
- Security team needs audit logs
- Legal team needs data retention controls

I'd involve these stakeholders early to ensure we're building the right thing."

**What makes this L7-appropriate**:
- System of systems thinking
- Organizational considerations
- Build vs buy analysis
- Cross-functional stakeholders
- Technical strategy over time

---

## 3️⃣ Calibration Guide

### How to Calibrate Your Responses

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CALIBRATION CHECKLIST                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  IF INTERVIEWING FOR L4:                                            │
│  □ Focus on getting a working design                                │
│  □ Don't over-engineer                                              │
│  □ It's okay to ask for hints                                       │
│  □ Show learning ability                                            │
│                                                                      │
│  IF INTERVIEWING FOR L5:                                            │
│  □ Drive the interview independently                                │
│  □ Cover all major areas                                            │
│  □ Do capacity estimation                                           │
│  □ Discuss trade-offs proactively                                   │
│                                                                      │
│  IF INTERVIEWING FOR L6:                                            │
│  □ Present multiple approaches                                      │
│  □ Go deep on complex topics                                        │
│  □ Consider operational concerns                                    │
│  □ Show expert-level knowledge                                      │
│                                                                      │
│  IF INTERVIEWING FOR L7:                                            │
│  □ Think about system of systems                                    │
│  □ Consider organizational impact                                   │
│  □ Discuss technical strategy                                       │
│  □ Show leadership in problem-solving                               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Common Mistakes by Level

**L4 mistakes**:
- Over-engineering (microservices for a simple problem)
- Pretending to know more than you do
- Not asking for help when stuck

**L5 mistakes**:
- Not driving the interview (waiting for questions)
- Missing major components
- Not doing capacity estimation
- Shallow trade-off discussion

**L6 mistakes**:
- Not presenting alternatives
- Missing operational concerns
- Not going deep enough
- Ignoring edge cases

**L7 mistakes**:
- Too focused on single system
- Not considering organizational impact
- Missing strategic perspective
- Not connecting to business outcomes

---

## 4️⃣ One Clean Mental Summary

Level expectations determine what interviewers look for. L4 should demonstrate basic competence and learning ability. L5 should drive independently and cover all areas with trade-offs. L6 should present multiple approaches with deep expertise and operational awareness. L7 should think about systems of systems with organizational and strategic considerations.

Calibrate your responses to your target level. Under-delivering means you're not ready for the level. Over-engineering means you might not be a good fit for the team's needs. Match your depth and breadth to what's expected at your level.

The key differentiator as you move up levels is not just technical depth, but breadth of consideration: from "does it work?" (L4) to "what are the trade-offs?" (L5) to "how does it operate?" (L6) to "how does it fit the organization?" (L7).

