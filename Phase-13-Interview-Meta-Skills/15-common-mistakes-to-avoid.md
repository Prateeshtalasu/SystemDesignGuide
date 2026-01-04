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

**Why it's bad**:
- Missed opportunity to clarify requirements
- May design for wrong scale or features
- Shows lack of senior-level thinking
- Interviewer wonders if you understand the problem

**How to fix**:
"Before I design, let me clarify requirements. What features are in scope? What's the expected scale? What are the latency requirements?"

**Time investment**: 5-7 minutes on requirements saves redesign later.

---

### Mistake 2: Not Clarifying Requirements

**What it looks like**:
Candidate assumes features, scale, and constraints without asking.

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

---

### Mistake 3: Ignoring Non-Functional Requirements

**What it looks like**:
Candidate designs features but never discusses scale, latency, or availability.

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

---

### Mistake 4: Over-Engineering

**What it looks like**:
Designing microservices, Kubernetes, and global distribution for a system that serves 1000 users.

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

---

### Mistake 5: Under-Engineering

**What it looks like**:
"We'll just use a single PostgreSQL database" for a system expecting 1 million QPS.

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

---

### Mistake 6: Not Discussing Trade-offs

**What it looks like**:
"I'll use Redis for caching." (No explanation of why, or what alternatives were considered.)

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

"I'll use Redis for caching because we need sub-millisecond latency and Redis supports rich data structures. The alternative was Memcached, which is simpler but lacks features we might need. The trade-off is Redis is more complex to operate."

---

### Mistake 7: Poor Time Management

**What it looks like**:
Spending 20 minutes on database schema, then rushing through everything else.

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

---

### Mistake 8: Not Drawing Diagrams

**What it looks like**:
Explaining the entire design verbally without using the whiteboard.

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

---

### Mistake 9: Not Asking Questions

**What it looks like**:
Candidate never checks in with interviewer, never asks for clarification.

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

---

### Mistake 10: Getting Defensive When Challenged

**What it looks like**:
Interviewer: "Why not use NoSQL here?"
Candidate: "SQL is fine. NoSQL has its own problems."

**Why it's bad**:
- Misses chance to show flexibility
- Interviewer may have been hinting at a better approach
- Shows inability to receive feedback
- Red flag for collaboration

**How to fix**:
Treat challenges as opportunities:
"Good question. Let me think about that. NoSQL would give us [benefits]. The trade-off is [downsides]. Given our requirements for [X], I still lean toward SQL, but I could see NoSQL working if [condition]. What's your thinking?"

---

### Mistake 11: Giving Vague Answers

**What it looks like**:
"We'd use caching to make it faster."
"We'd scale horizontally."
"We'd use a message queue."

**Why it's bad**:
- Doesn't demonstrate understanding
- Interviewer can't evaluate depth
- Sounds like memorized buzzwords
- Doesn't differentiate from other candidates

**How to fix**:
Be specific:
- "We'd use Redis as a cache-aside cache with 1-hour TTL. I'd expect 95% hit rate based on access patterns."
- "We'd scale horizontally by adding more stateless API servers behind the load balancer. Auto-scaling would trigger at 70% CPU."
- "We'd use Kafka for async processing of notifications. Partitioned by user_id for ordering."

---

### Mistake 12: Not Handling Failure Scenarios

**What it looks like**:
Design only covers the happy path. No discussion of what happens when things fail.

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

"If the primary database fails, we have automatic failover to the replica. Detection is via health checks every 10 seconds. Failover takes 1-2 minutes. During this time, writes fail but reads continue from other replicas."

---

### Mistake 13: Memorizing Without Understanding

**What it looks like**:
Can recite that "Kafka is used for event streaming" but can't explain when to use it vs RabbitMQ, or how it handles ordering.

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

---

### Mistake 14: Ignoring the Interviewer

**What it looks like**:
Candidate talks continuously without pausing, ignores interviewer's attempts to interject.

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

---

### Mistake 15: Not Summarizing

**What it looks like**:
Design ends abruptly without summary or wrap-up.

**Why it's bad**:
- Misses chance to reinforce key points
- Interviewer may forget earlier parts
- No closure to the discussion
- Seems incomplete

**How to fix**:
Always end with a summary:
"To summarize, we designed a [system] that handles [scale]. Key components are [A, B, C]. The main trade-off is [X], which we accepted because [reason]. For monitoring, I'd track [metrics]. Future improvements would include [ideas]."

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

