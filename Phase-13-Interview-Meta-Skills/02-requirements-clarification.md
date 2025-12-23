# Requirements Clarification

## 0️⃣ Prerequisites

Before diving into requirements clarification, you should understand:

- **Problem Approach Framework**: The overall structure of a system design interview (covered in Topic 1 of this phase)
- **Basic System Components**: Familiarity with databases, caches, load balancers, and APIs (covered in Phases 1-6)
- **Non-Functional Requirements Concepts**: Understanding of scalability, availability, and consistency (covered in Phase 1)

Quick refresher: Requirements clarification is the first phase of a system design interview, typically lasting 5-10 minutes. Its purpose is to scope the problem, uncover constraints, and demonstrate senior-level thinking before diving into the solution.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

System design interview prompts are intentionally vague. "Design Twitter," "Design a URL shortener," or "Design a chat application" are not specifications. They're starting points for a conversation.

The vagueness is deliberate. Interviewers want to see:

1. **Can you identify ambiguity?** Real-world projects start with unclear requirements.
2. **Can you ask the right questions?** Senior engineers don't wait to be told everything.
3. **Can you prioritize?** Not everything can be built. What matters most?
4. **Can you scope appropriately?** Neither over-engineering nor under-engineering.

### What Breaks Without Requirements Clarification

**Scenario 1: The Wrong Scale**

Candidate designs a beautiful single-server architecture for "Design a notification system." At the end, the interviewer reveals they wanted to handle 1 billion notifications per day. The entire design is useless.

**Scenario 2: The Wrong Features**

Candidate spends 30 minutes designing a sophisticated search algorithm for "Design Twitter." The interviewer wanted to focus on the timeline feed. The search work was wasted effort.

**Scenario 3: The Wrong Consistency Model**

Candidate designs a strongly consistent system for "Design a social feed." The interviewer asks, "Why did you choose strong consistency? We said eventual consistency is fine." The candidate never asked.

**Scenario 4: The Missing Constraint**

Candidate designs a cloud-native solution for "Design a data pipeline." The interviewer reveals the company has strict data residency requirements and can't use public cloud. The design violates a fundamental constraint.

### Real Examples of the Problem

**Example 1: Google Interview Gone Wrong**

A candidate designing Google Docs jumped straight into the CRDT (Conflict-free Replicated Data Type) implementation. The interviewer had to interrupt: "Wait, are we supporting real-time collaboration?" The candidate assumed yes. The interviewer wanted to focus on document storage and versioning first. The candidate lost 15 minutes and had to restart.

**Example 2: Amazon Interview Misalignment**

A candidate designing a recommendation system focused entirely on the ML algorithm. The interviewer wanted to discuss the data pipeline, feature store, and serving infrastructure. The candidate's ML expertise was impressive but irrelevant to what the interviewer was evaluating.

**Example 3: Meta Interview Feature Creep**

A candidate designing Facebook Messenger kept adding features: read receipts, typing indicators, message reactions, stories, payments. The interviewer finally said, "Let's just focus on sending and receiving messages." The candidate had wasted 10 minutes on features that were out of scope.

---

## 2️⃣ Intuition and Mental Model

### The House Building Analogy

Imagine you're an architect and a client says, "Build me a house."

A junior architect immediately starts drawing blueprints. They assume a 3-bedroom suburban home because that's what they know.

A senior architect asks questions:

- "How many people will live here?" (Scale)
- "What's your budget?" (Constraints)
- "Do you need a home office?" (Functional requirements)
- "Is energy efficiency important?" (Non-functional requirements)
- "Are you in a flood zone?" (Environmental constraints)
- "How long do you plan to live here?" (Future considerations)

The answers completely change the design:
- 1 person vs 6 people → Studio vs 5-bedroom
- $200K vs $2M budget → Modest vs luxury
- Work from home → Dedicated office space
- Flood zone → Elevated foundation
- 5 years vs forever → Flexibility vs permanence

System design is identical. "Design Twitter" could mean:
- Twitter for 1000 users (startup MVP)
- Twitter for 100 million users (growth stage)
- Twitter for 500 million users (current scale)

Each requires fundamentally different architecture.

### The Funnel Model

Think of requirements clarification as a funnel that narrows the problem space:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    REQUIREMENTS FUNNEL                               │
└─────────────────────────────────────────────────────────────────────┘

                         "Design Twitter"
                               │
        ┌──────────────────────┴──────────────────────┐
        │                                             │
        │    INFINITE POSSIBLE INTERPRETATIONS        │
        │    - Scale: 1K to 1B users                  │
        │    - Features: Tweets, DMs, Spaces, Ads...  │
        │    - Consistency: Strong to eventual        │
        │    - Geography: Single region to global     │
        │                                             │
        └──────────────────────┬──────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  CLARIFY FUNCTIONAL │
                    │  "Core features?"   │
                    └──────────┬──────────┘
                               │
                      ┌────────▼────────┐
                      │ CLARIFY SCALE   │
                      │ "How many DAU?" │
                      └────────┬────────┘
                               │
                        ┌──────▼──────┐
                        │CLARIFY NFRs │
                        │"Latency?"   │
                        └──────┬──────┘
                               │
                          ┌────▼────┐
                          │CONSTRAIN│
                          │"Budget?"│
                          └────┬────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  SCOPED PROBLEM     │
                    │                     │
                    │  - 200M DAU         │
                    │  - Tweets + Feed    │
                    │  - Eventual consist │
                    │  - p99 < 200ms      │
                    │  - Global           │
                    └─────────────────────┘
```

Each question narrows the funnel until you have a well-defined problem to solve.

---

## 3️⃣ How It Works Internally

Requirements clarification has three main categories:

### Category 1: Functional Requirements

**Definition**: What the system must DO. The features and capabilities.

**Structure**:

```
FUNCTIONAL REQUIREMENTS
├── Core Features (Must Have)
│   └── Essential for MVP, system is useless without these
├── Extended Features (Should Have)
│   └── Important but can be added later
├── Nice-to-Have Features (Could Have)
│   └── Would be great but not critical
└── Out of Scope (Won't Have)
    └── Explicitly excluded from this design
```

**Questions to ask**:

```
CORE FEATURES:
- "What are the absolute must-have features?"
- "What's the primary user action?"
- "What problem are we solving for the user?"

FEATURE BOUNDARIES:
- "Should we support X feature?" (for each major feature)
- "Is Y in scope or out of scope?"
- "Are we building for consumers, businesses, or both?"

USER TYPES:
- "Who are the users? End users? Admins? Both?"
- "Are there different user roles with different capabilities?"
- "Is this internal or external facing?"
```

**Example: Design Instagram**

Candidate: "For functional requirements, let me confirm the core features. Are we focusing on:
1. Photo/video upload and storage
2. Feed showing posts from followed users
3. Like and comment functionality
4. User profiles and follow system

Are DMs, Stories, Reels, and Explore in scope?"

Interviewer: "Let's focus on 1-4. DMs and Stories are out of scope."

Now the candidate knows exactly what to design.

### Category 2: Non-Functional Requirements (NFRs)

**Definition**: How the system must PERFORM. Quality attributes.

**The Big Five NFRs**:

```
NON-FUNCTIONAL REQUIREMENTS
├── 1. SCALE
│   ├── Users (DAU, MAU)
│   ├── Requests (QPS, peak QPS)
│   ├── Data (storage volume, growth rate)
│   └── Geographic distribution
│
├── 2. PERFORMANCE
│   ├── Latency (p50, p99, p999)
│   ├── Throughput (requests/second)
│   └── Response time targets
│
├── 3. AVAILABILITY
│   ├── Uptime target (99.9%, 99.99%)
│   ├── Acceptable downtime
│   └── Disaster recovery requirements
│
├── 4. CONSISTENCY
│   ├── Strong consistency (reads see latest write)
│   ├── Eventual consistency (reads may be stale)
│   └── Causal consistency (related operations ordered)
│
└── 5. DURABILITY
    ├── Data loss tolerance (zero loss? some acceptable?)
    ├── Backup requirements
    └── Retention policies
```

**Questions to ask**:

```
SCALE:
- "What's the expected number of daily active users?"
- "What's the read/write ratio?"
- "How much data per user/entity?"
- "Is this single region or global?"

PERFORMANCE:
- "What latency is acceptable? p99 target?"
- "Are there any hard real-time requirements?"
- "What's more important: latency or throughput?"

AVAILABILITY:
- "What's the availability target? 99.9%? 99.99%?"
- "Is some downtime acceptable for maintenance?"
- "What's the disaster recovery requirement?"

CONSISTENCY:
- "Is strong consistency required?"
- "Is it okay if users see slightly stale data?"
- "Are there any operations that MUST be strongly consistent?"

DURABILITY:
- "Can we ever lose data, or is zero data loss required?"
- "What's the data retention policy?"
- "Do we need point-in-time recovery?"
```

**Example: Design a Payment System**

Candidate: "For non-functional requirements:

Scale: How many transactions per day? Per second at peak?

Performance: What's the acceptable latency for a payment? I assume sub-second.

Availability: Payment systems typically need 99.99% uptime. Is that our target?

Consistency: I assume strong consistency is required. We can't have double charges or lost payments.

Durability: Zero data loss, I assume. Every transaction must be persisted."

Interviewer: "Yes to all. Let's say 10 million transactions/day, 500 TPS at peak, p99 under 500ms."

Now the candidate knows this is a high-consistency, moderate-scale system.

### Category 3: Constraints and Assumptions

**Definition**: Limitations and context that affect design decisions.

**Types of constraints**:

```
CONSTRAINTS
├── TECHNICAL
│   ├── Existing infrastructure
│   ├── Technology stack requirements
│   ├── Integration requirements
│   └── Legacy system compatibility
│
├── BUSINESS
│   ├── Budget
│   ├── Timeline
│   ├── Team size and expertise
│   └── Regulatory requirements
│
├── OPERATIONAL
│   ├── Deployment environment (cloud, on-prem, hybrid)
│   ├── Monitoring and observability requirements
│   └── On-call and support model
│
└── ENVIRONMENTAL
    ├── Data residency requirements
    ├── Compliance (GDPR, HIPAA, PCI-DSS)
    └── Security requirements
```

**Questions to ask**:

```
TECHNICAL:
- "Are we integrating with existing systems?"
- "Any technology preferences or restrictions?"
- "Is this greenfield or brownfield?"

BUSINESS:
- "Are there cost constraints we should consider?"
- "Any compliance requirements (GDPR, HIPAA)?"
- "What's the timeline for MVP vs full launch?"

OPERATIONAL:
- "Cloud, on-prem, or hybrid?"
- "Any specific cloud provider requirements?"
- "What's the deployment model?"
```

**Example: Design a Healthcare Data Platform**

Candidate: "Before I design, I need to understand constraints:

Compliance: Is this handling PHI (Protected Health Information)? If so, we need HIPAA compliance.

Data residency: Any requirements on where data is stored? Some healthcare data can't leave certain jurisdictions.

Integration: Are we integrating with existing EHR (Electronic Health Record) systems?

Security: What authentication is required? SSO? MFA?"

Interviewer: "Yes to HIPAA. Data must stay in US. We're integrating with Epic and Cerner. SSO with MFA required."

These constraints fundamentally shape the architecture.

---

## 4️⃣ Simulation: Requirements Clarification in Action

Let's walk through a complete requirements clarification for "Design Uber."

### The Interview Begins

**Interviewer**: "Design Uber."

**Candidate**: "Great, I'd like to start by clarifying requirements to make sure I'm solving the right problem. I'll ask about functional requirements, scale, and constraints. Should take about 5-7 minutes."

**Interviewer**: "Go ahead."

### Functional Requirements

**Candidate**: "For functional requirements, I want to confirm the core user flows. I see two main actors: riders and drivers.

For riders:
1. Request a ride from point A to point B
2. See available drivers nearby
3. Get matched with a driver
4. Track driver location in real-time
5. Pay for the ride

For drivers:
1. Go online/offline
2. See ride requests
3. Accept or decline rides
4. Navigate to pickup and dropoff
5. Receive payment

Are these the core flows we should focus on?"

**Interviewer**: "Yes, that covers it."

**Candidate**: "Are the following in scope or out of scope?
- Surge pricing
- Ride scheduling (book for later)
- Ride pooling (UberPool)
- Driver ratings and reviews
- Ride history and receipts"

**Interviewer**: "Let's include surge pricing. The others are out of scope."

**Candidate**: "Got it. So core ride-hailing with surge pricing. No pooling, scheduling, or ratings."

### Non-Functional Requirements

**Candidate**: "For scale, what geography are we targeting?"

**Interviewer**: "Let's say a major city like New York."

**Candidate**: "For a city like NYC, I'll estimate:
- 500,000 daily rides
- 50,000 concurrent drivers at peak
- 100,000 concurrent rider requests at peak

Does that sound reasonable?"

**Interviewer**: "Yes, that's a good estimate."

**Candidate**: "For latency, ride matching is time-sensitive. What's acceptable?
- Time from request to match: I'd guess under 30 seconds
- Location update frequency: Every few seconds
- ETA accuracy: Within a minute or two"

**Interviewer**: "Those are reasonable targets."

**Candidate**: "For consistency:
- Ride matching must be consistent. We can't match one driver to two riders.
- Location updates can be eventually consistent. A few seconds of staleness is okay.
- Payment must be strongly consistent.

Does that align with your expectations?"

**Interviewer**: "Yes, exactly."

**Candidate**: "Availability target? I assume 99.9% or higher. Uber being down means people are stranded."

**Interviewer**: "99.9% is our target."

### Constraints

**Candidate**: "Any constraints I should know about?
- Existing infrastructure or technology preferences?
- Cost considerations?
- Regulatory requirements for ride-hailing?"

**Interviewer**: "Assume we can use any technology. No specific cost constraints. Assume we handle regulatory requirements separately."

### Summary

**Candidate**: "Let me summarize what we're building:

**Functional scope**:
- Rider requests ride, gets matched with driver
- Real-time tracking
- Surge pricing
- Payment processing
- Out of scope: pooling, scheduling, ratings

**Scale**:
- 500K daily rides in NYC
- 50K concurrent drivers, 100K concurrent requests at peak
- Global expansion later, but design for single city first

**Performance**:
- Match within 30 seconds
- Location updates every 3-5 seconds
- p99 latency under 500ms for API calls

**Consistency**:
- Strong for matching and payment
- Eventual for location updates

**Availability**: 99.9%

Does this capture the requirements correctly?"

**Interviewer**: "Yes, that's a great summary. Let's proceed with the design."

The candidate has spent 5-7 minutes but now has crystal clear scope. Every design decision can reference these requirements.

---

## 5️⃣ How Engineers Actually Use This in Production

### Real Interview Experiences

**Netflix Staff Engineer (2023)**:
"I was asked to design a video recommendation system. I spent 8 minutes on requirements, which felt like a lot. But it paid off. I discovered they wanted near-real-time recommendations (update as you watch), not batch recommendations (update daily). That completely changed the architecture from a batch pipeline to a streaming system."

**Stripe Senior Engineer (2022)**:
"Design a payment system. I asked about consistency requirements and learned they wanted exactly-once semantics for payment processing. That led me to discuss idempotency keys, two-phase commits, and saga patterns. If I hadn't asked, I might have designed a simpler at-least-once system."

**Google L6 (2023)**:
"Design Google Maps directions. I asked 'Are we optimizing for fastest route, shortest distance, or fewest turns?' The interviewer said 'All three, user can choose.' That meant I needed a flexible routing algorithm that could optimize for different objectives. One question changed the entire design."

### The Senior vs Junior Difference

**Junior approach**:
- Asks 1-2 surface-level questions
- Accepts first answer without probing
- Misses non-functional requirements
- Doesn't summarize or confirm

**Senior approach**:
- Systematic coverage of functional, NFR, and constraints
- Probes deeper on ambiguous answers
- Explicitly confirms out-of-scope items
- Summarizes and gets confirmation before proceeding

### Template for Requirements Clarification

Here's a template you can adapt:

```
OPENING:
"Before I dive into the design, I'd like to spend a few minutes 
clarifying requirements. This will help me design the right system."

FUNCTIONAL:
"Let me confirm the core features..."
"Are the following in scope: [list features]?"
"What's explicitly out of scope?"

SCALE:
"What's the expected scale? Users, requests, data volume?"
"Is this single region or global?"
"What's the read/write ratio?"

PERFORMANCE:
"What latency is acceptable?"
"Any real-time requirements?"

CONSISTENCY:
"Is strong consistency required, or is eventual consistency acceptable?"
"Are there specific operations that must be strongly consistent?"

AVAILABILITY:
"What's the availability target?"
"Any disaster recovery requirements?"

CONSTRAINTS:
"Any technology constraints or preferences?"
"Any compliance requirements?"
"Cost constraints?"

SUMMARY:
"Let me summarize what we're building: [recap all requirements]"
"Does this capture the requirements correctly?"
```

---

## 6️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Pitfall 1: Asking Too Few Questions

**What happens**: You make assumptions that turn out to be wrong.

**Example**: Assuming a chat system needs end-to-end encryption when the interviewer wanted a simple internal team chat.

**Fix**: Use the template. Cover functional, NFR, and constraints systematically.

### Pitfall 2: Asking Too Many Questions

**What happens**: You spend 15 minutes on requirements and run out of time for the design.

**Example**: Asking about every possible edge case, regulatory detail, and future feature.

**Fix**: Focus on questions that materially affect architecture. Skip details that don't change the design.

### Pitfall 3: Not Listening to Answers

**What happens**: You ask good questions but don't incorporate the answers into your design.

**Example**: Interviewer says "eventual consistency is fine" but you design a strongly consistent system anyway.

**Fix**: Take notes. Reference requirements throughout your design. "As we discussed, eventual consistency is acceptable, so I'll use async replication here."

### Pitfall 4: Not Summarizing

**What happens**: You and the interviewer have different understandings of the requirements.

**Example**: You think you're designing for 1 million users, interviewer thinks 1 billion.

**Fix**: Always summarize requirements before proceeding. Get explicit confirmation.

### Pitfall 5: Treating Requirements as Fixed

**What happens**: You don't revisit requirements when you discover design challenges.

**Example**: You realize strong consistency at scale is very expensive, but don't ask if eventual consistency might be acceptable.

**Fix**: Requirements can be negotiated. "I'm finding that strong consistency at this scale is challenging. Would eventual consistency with a 5-second window be acceptable?"

### Pitfall 6: Ignoring Non-Functional Requirements

**What happens**: Your design works functionally but fails at scale, latency, or availability.

**Example**: Beautiful microservices architecture that has 2-second latency because you didn't consider the latency target.

**Fix**: NFRs often matter more than functional requirements. A system that's slow or unreliable is worse than one missing features.

---

## 7️⃣ When NOT to Spend Time on Requirements

### Scenario 1: Very Specific Prompts

If the interviewer gives a detailed prompt with explicit requirements, don't re-ask everything.

**Example**: "Design a URL shortener that handles 100M URLs per day, with 7-character short codes, 301 redirects, and basic analytics."

**Action**: Acknowledge the requirements, ask 1-2 clarifying questions, then proceed.

### Scenario 2: Interviewer Wants to Jump In

Some interviewers prefer to give requirements incrementally as you design.

**Signs**: They seem impatient with questions, they say "let's see where the design goes," they add requirements as you draw.

**Action**: Start with minimal clarification, be ready to adapt as new requirements emerge.

### Scenario 3: Time Pressure

If you're told "we only have 30 minutes," you can't spend 10 on requirements.

**Action**: Ask the 3 most critical questions (scale, core features, consistency), then proceed.

---

## 8️⃣ Comparison: Good vs Bad Requirements Clarification

### Bad Example: Design a Chat Application

**Candidate**: "Should it support group chats?"

**Interviewer**: "Yes."

**Candidate**: "Okay, let me start designing..."

**Problems**:
- Only one question asked
- No scale discussion
- No latency requirements
- No consistency discussion
- No summary

### Good Example: Design a Chat Application

**Candidate**: "I'd like to clarify requirements first. For functional scope:
- One-on-one messaging: definitely in scope
- Group messaging: in scope?
- Read receipts and typing indicators: in scope?
- File/image sharing: in scope?
- Search message history: in scope?"

**Interviewer**: "One-on-one and groups, yes. Read receipts yes. Files yes. Search is out of scope."

**Candidate**: "For scale:
- How many daily active users?
- Messages per day?
- Average group size?"

**Interviewer**: "100 million DAU, 10 billion messages per day, average group is 10 people."

**Candidate**: "For performance:
- What's acceptable message delivery latency? I assume near real-time, under 1 second.
- Is message ordering important? Should messages always appear in send order?"

**Interviewer**: "Yes, under 500ms. Yes, ordering matters within a conversation."

**Candidate**: "For consistency:
- Is it okay if users briefly see messages in different orders before they converge?
- Is at-least-once delivery acceptable, or do we need exactly-once?"

**Interviewer**: "Brief inconsistency is okay. At-least-once is fine, we'll deduplicate on client."

**Candidate**: "For availability, I assume 99.9% uptime minimum. For durability, messages should never be lost once sent."

**Interviewer**: "Correct."

**Candidate**: "Let me summarize:
- 1:1 and group chat with read receipts and file sharing
- 100M DAU, 10B messages/day
- Sub-500ms delivery, ordered within conversation
- Eventually consistent, at-least-once delivery
- 99.9% availability, zero message loss
- Search is out of scope

Is this correct?"

**Interviewer**: "Yes, let's proceed."

This candidate has a clear, confirmed scope and can make confident design decisions.

---

## 9️⃣ Interview Follow-up Questions WITH Answers

### Q1: "Why do you spend so much time on requirements?"

**Answer**: "Requirements clarification serves multiple purposes. First, it ensures I'm solving the right problem. A chat system for 1000 users is fundamentally different from one for 100 million. Second, it demonstrates senior-level thinking. Junior engineers jump to solutions. Senior engineers ensure they understand the problem first. Third, it gives me anchors for design decisions. When I choose eventual consistency, I can reference 'as we discussed, sub-second latency matters more than immediate consistency.' Finally, it builds rapport with the interviewer. We're collaborating on defining the problem together."

### Q2: "What if the interviewer says 'just make assumptions'?"

**Answer**: "I'd make reasonable assumptions and state them explicitly. 'I'll assume we're designing for 10 million DAU, eventual consistency is acceptable, and we need 99.9% availability. Please let me know if any of these don't match your expectations.' This shows I can make decisions while remaining open to course correction. I'd also ask the 1-2 most critical questions that would fundamentally change the design, like 'Is this a real-time system or batch processing?'"

### Q3: "How do you prioritize which questions to ask?"

**Answer**: "I prioritize questions that would fundamentally change the architecture. Scale is almost always critical because a single-server design is completely different from a distributed system. Consistency requirements matter because they affect database choice and replication strategy. Real-time vs batch processing changes everything. I skip questions about minor features or details that don't affect the high-level architecture. For example, 'Should usernames be case-sensitive?' doesn't change the system design."

### Q4: "What if you realize mid-design that you missed a requirement?"

**Answer**: "I'd acknowledge it immediately: 'I realize I should have asked about X earlier. Let me clarify now.' Then I'd ask the question and adjust the design if needed. This is actually a positive signal. It shows I'm continuously evaluating whether my design meets requirements, and I'm not afraid to course-correct. The worst thing would be to silently continue with a flawed design."

### Q5: "How do you handle conflicting requirements?"

**Answer**: "I'd surface the conflict explicitly: 'You mentioned we need both sub-100ms latency and strong consistency across regions. These are in tension because strong consistency requires coordination that adds latency. Can we prioritize one over the other, or is there a specific scenario where we can relax one requirement?' This shows I understand the tradeoffs and can facilitate a conversation about priorities. Often the interviewer will clarify which matters more, or we'll find a middle ground."

### Q6: "What's the difference between L4 and L6 requirements clarification?"

**Answer**: 
- **L4**: Asks basic questions about features and scale. May miss non-functional requirements. Accepts answers at face value.
- **L5**: Systematic coverage of functional and non-functional requirements. Probes deeper on ambiguous answers. Summarizes before proceeding.
- **L6**: All of the above, plus identifies potential conflicts between requirements, suggests tradeoffs proactively, and frames requirements in terms of business impact. Might say, 'Strong consistency here means we're choosing correctness over availability. Is that the right tradeoff for this business?'

---

## 🔟 One Clean Mental Summary

Requirements clarification is the foundation of a successful system design interview. In 5-10 minutes, you transform a vague prompt like "Design Twitter" into a well-defined problem with clear scope, scale, and constraints.

The key categories are: functional requirements (what features), non-functional requirements (scale, latency, consistency, availability), and constraints (technology, compliance, budget).

The senior approach is systematic: cover all categories, probe deeper on ambiguous answers, and summarize before proceeding. This demonstrates that you understand real-world engineering starts with understanding the problem, not jumping to solutions.

A well-clarified requirement set is like a compass. Throughout the design, you can reference it to justify decisions: "We chose eventual consistency because, as we discussed, sub-second latency is more important than immediate consistency for this use case."

---

## Quick Reference: Requirements Checklist

```
┌─────────────────────────────────────────────────────────────────────┐
│              REQUIREMENTS CLARIFICATION CHECKLIST                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  FUNCTIONAL REQUIREMENTS                                            │
│  □ Core features (must-have for MVP)                                │
│  □ Extended features (nice-to-have)                                 │
│  □ Out of scope (explicitly excluded)                               │
│  □ User types (end users, admins, internal, external)               │
│                                                                      │
│  SCALE                                                              │
│  □ Daily/monthly active users                                       │
│  □ Requests per second (read and write)                             │
│  □ Data volume (per entity, total, growth rate)                     │
│  □ Geographic distribution (single region, multi-region, global)    │
│                                                                      │
│  PERFORMANCE                                                        │
│  □ Latency targets (p50, p99)                                       │
│  □ Throughput requirements                                          │
│  □ Real-time vs batch processing                                    │
│                                                                      │
│  CONSISTENCY                                                        │
│  □ Strong vs eventual consistency                                   │
│  □ Which operations require strong consistency                      │
│  □ Acceptable staleness window                                      │
│                                                                      │
│  AVAILABILITY                                                       │
│  □ Uptime target (99.9%, 99.99%, 99.999%)                          │
│  □ Acceptable downtime for maintenance                              │
│  □ Disaster recovery requirements                                   │
│                                                                      │
│  DURABILITY                                                         │
│  □ Data loss tolerance                                              │
│  □ Backup and recovery requirements                                 │
│  □ Retention policies                                               │
│                                                                      │
│  CONSTRAINTS                                                        │
│  □ Technology preferences or restrictions                           │
│  □ Existing systems to integrate with                               │
│  □ Compliance requirements (GDPR, HIPAA, PCI-DSS)                  │
│  □ Budget or cost constraints                                       │
│  □ Team size and expertise                                          │
│                                                                      │
│  SUMMARY                                                            │
│  □ Recap all requirements                                           │
│  □ Get explicit confirmation from interviewer                       │
│  □ Note any assumptions made                                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Common Questions by System Type

### For Real-Time Systems (Chat, Gaming, Trading)

```
- What's the acceptable message/event latency?
- Is ordering guaranteed?
- What happens if a message is delayed or lost?
- How do we handle offline users?
- Is there a need for message persistence?
```

### For Data-Intensive Systems (Analytics, ML, Search)

```
- What's the data volume and growth rate?
- Real-time or batch processing?
- What's the query pattern (point lookups vs aggregations)?
- How fresh does the data need to be?
- What's the retention policy?
```

### For E-Commerce/Transactional Systems

```
- What's the consistency requirement for inventory?
- How do we handle payment failures?
- Is there a need for exactly-once processing?
- What's the peak traffic pattern (flash sales)?
- International or domestic?
```

### For Social/Feed Systems

```
- What's the read/write ratio?
- How fresh does the feed need to be?
- Is content ranking required?
- How do we handle viral content (hot partitions)?
- Is there a need for real-time updates (push) or pull-based refresh?
```

### For Storage/File Systems

```
- What's the file size distribution?
- What's the access pattern (write once, read many)?
- Is versioning required?
- What's the durability requirement?
- Is there a need for sharing/permissions?
```

