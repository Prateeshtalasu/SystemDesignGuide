# Mock Interview Structure

## 0️⃣ Prerequisites

Before diving into mock interviews, you should understand:

- **Problem Approach Framework**: How to structure your interview (covered in Topic 1)
- **All Previous Topics**: Core system design knowledge and interview skills (covered in Topics 1-13)

Quick refresher: Mock interviews are the most effective way to prepare for system design interviews. Reading about system design is not the same as doing it under time pressure. Mock interviews build the muscle memory needed to perform well when it matters.

---

## 1️⃣ Why Mock Interviews Matter

### The Practice Gap

Many candidates study system design extensively but struggle in actual interviews because:

1. **Time pressure**: 45 minutes feels different than unlimited study time
2. **Communication**: Explaining while thinking is a skill that requires practice
3. **Adaptability**: Responding to unexpected questions requires experience
4. **Nerves**: Performance anxiety affects even well-prepared candidates
5. **Feedback blind spots**: You don't know what you don't know

### The Research

Studies show that deliberate practice with feedback is the most effective way to improve performance. For system design interviews:

- Reading about system design: 20% retention
- Watching videos: 30% retention
- Doing mock interviews: 70% retention
- Doing mock interviews with feedback: 90% retention

---

## 2️⃣ Mock Interview Structure

### The 45-Minute Format

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MOCK INTERVIEW STRUCTURE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  0:00 - 0:02   PROBLEM INTRODUCTION                                 │
│  ─────────────────────────────────                                  │
│  Interviewer presents the problem                                   │
│  Candidate listens and takes notes                                  │
│                                                                      │
│  0:02 - 0:08   REQUIREMENTS CLARIFICATION                           │
│  ─────────────────────────────────────                              │
│  Functional requirements                                            │
│  Non-functional requirements (scale, latency)                       │
│  Constraints and assumptions                                        │
│                                                                      │
│  0:08 - 0:12   CAPACITY ESTIMATION                                  │
│  ────────────────────────────────                                   │
│  Traffic estimation (QPS)                                           │
│  Storage estimation                                                 │
│  Bandwidth estimation                                               │
│                                                                      │
│  0:12 - 0:28   HIGH-LEVEL DESIGN                                    │
│  ───────────────────────────────                                    │
│  Draw architecture diagram                                          │
│  Explain each component                                             │
│  Walk through main data flows                                       │
│                                                                      │
│  0:28 - 0:40   DEEP DIVE                                            │
│  ───────────────────────                                            │
│  Focus on 1-2 complex components                                    │
│  Discuss algorithms, data structures                                │
│  Handle edge cases and failures                                     │
│                                                                      │
│  0:40 - 0:45   WRAP-UP                                              │
│  ─────────────────────                                              │
│  Discuss trade-offs                                                 │
│  Mention monitoring and operations                                  │
│  Future improvements                                                │
│                                                                      │
│  POST-INTERVIEW   FEEDBACK (10-15 min)                              │
│  ─────────────────────────────────────                              │
│  What went well                                                     │
│  Areas for improvement                                              │
│  Specific actionable feedback                                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Roles and Responsibilities

**As the Candidate**:
- Treat it like a real interview
- Don't look at notes (unless you would in a real interview)
- Ask clarifying questions
- Think out loud
- Manage your time

**As the Interviewer**:
- Present the problem clearly
- Answer clarifying questions (don't give too much away)
- Ask follow-up questions
- Probe weak areas
- Keep track of time
- Take notes for feedback

---

## 3️⃣ Finding Practice Partners

### Option 1: Peer Practice

**Where to find peers**:
- Colleagues also preparing for interviews
- Online communities (Blind, Reddit r/cscareerquestions)
- Discord servers (System Design, Tech Interview)
- Local meetup groups
- University alumni networks

**How to structure peer practice**:
```
SESSION STRUCTURE (2 hours):

Hour 1:
- Person A interviews Person B (45 min)
- Feedback discussion (15 min)

Hour 2:
- Person B interviews Person A (45 min)
- Feedback discussion (15 min)

FREQUENCY: 2-3 times per week
```

**Pros**: Free, mutual benefit, flexible scheduling
**Cons**: Peer may not be experienced interviewer, quality varies

### Option 2: Professional Platforms

**Pramp** (free):
- Peer matching platform
- Structured mock interviews
- Both practice interviewing and being interviewed
- Good for general practice

**Interviewing.io** (paid):
- Anonymous practice with real engineers
- Engineers from top companies
- Detailed feedback
- Good for realistic experience

**Exponent** (paid):
- Structured courses with mock interviews
- Company-specific preparation
- Good for targeted prep

**How to use platforms effectively**:
```
RECOMMENDED APPROACH:

Weeks 1-2: Pramp (free)
- Get comfortable with format
- Practice basic problems
- Learn from peer feedback

Weeks 3-4: Interviewing.io (paid)
- Get feedback from experienced interviewers
- Identify blind spots
- Practice harder problems

Week 5+: Mix of both
- Continue practicing
- Focus on weak areas
```

### Option 3: Self-Practice

**When peers aren't available**:
- Record yourself (video or audio)
- Use a timer strictly
- Review your recording critically
- Compare to example solutions

**Self-practice structure**:
```
SELF-PRACTICE SESSION (1 hour):

1. Set timer for 45 minutes
2. Pick a random problem
3. Design out loud (record yourself)
4. Stop when timer ends
5. Review recording (15 min)
   - Did you cover all areas?
   - Was your communication clear?
   - Did you manage time well?
6. Compare to reference solutions
```

---

## 4️⃣ Practice Problems by Difficulty

### Beginner Problems (Start Here)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BEGINNER PROBLEMS                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. URL Shortener (bit.ly)                                          │
│     Focus: Basic CRUD, ID generation, caching                       │
│                                                                      │
│  2. Pastebin                                                        │
│     Focus: Storage, expiration, basic scaling                       │
│                                                                      │
│  3. Rate Limiter                                                    │
│     Focus: Algorithms, distributed systems basics                   │
│                                                                      │
│  4. Key-Value Store                                                 │
│     Focus: Storage, consistency, basic distribution                 │
│                                                                      │
│  5. Unique ID Generator                                             │
│     Focus: Distributed coordination, ordering                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Intermediate Problems

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INTERMEDIATE PROBLEMS                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Twitter/Social Feed                                             │
│     Focus: Fan-out, caching, real-time updates                      │
│                                                                      │
│  2. Instagram/Photo Sharing                                         │
│     Focus: Media storage, CDN, feed generation                      │
│                                                                      │
│  3. Chat System (WhatsApp)                                          │
│     Focus: Real-time messaging, presence, delivery                  │
│                                                                      │
│  4. Notification System                                             │
│     Focus: Multi-channel, reliability, prioritization               │
│                                                                      │
│  5. Search Autocomplete                                             │
│     Focus: Trie, ranking, real-time suggestions                     │
│                                                                      │
│  6. Web Crawler                                                     │
│     Focus: Distributed processing, politeness, dedup                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Advanced Problems

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ADVANCED PROBLEMS                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. YouTube/Video Streaming                                         │
│     Focus: Encoding, CDN, adaptive streaming                        │
│                                                                      │
│  2. Uber/Ride Sharing                                               │
│     Focus: Location, matching, real-time updates                    │
│                                                                      │
│  3. Google Docs/Collaborative Editing                               │
│     Focus: Conflict resolution, real-time sync                      │
│                                                                      │
│  4. Distributed Cache (Redis)                                       │
│     Focus: Consistency, partitioning, replication                   │
│                                                                      │
│  5. Search Engine (Google)                                          │
│     Focus: Indexing, ranking, distributed search                    │
│                                                                      │
│  6. Payment System (Stripe)                                         │
│     Focus: Consistency, idempotency, security                       │
│                                                                      │
│  7. Stock Exchange                                                  │
│     Focus: Matching engine, ordering, latency                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5️⃣ Giving and Receiving Feedback

### Feedback Framework

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FEEDBACK FRAMEWORK                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  STRUCTURE (SBI Model)                                              │
│  ─────────────────────                                              │
│  S - Situation: "When you were discussing the database..."          │
│  B - Behavior: "You didn't mention sharding..."                     │
│  I - Impact: "This made it seem like you hadn't considered scale"   │
│                                                                      │
│  AREAS TO EVALUATE                                                  │
│  ─────────────────                                                  │
│  □ Requirements clarification                                       │
│  □ Capacity estimation                                              │
│  □ High-level design completeness                                   │
│  □ Deep dive depth                                                  │
│  □ Trade-off discussion                                             │
│  □ Communication clarity                                            │
│  □ Time management                                                  │
│  □ Handling of questions                                            │
│                                                                      │
│  FEEDBACK BALANCE                                                   │
│  ────────────────                                                   │
│  - Start with what went well (2-3 points)                           │
│  - Areas for improvement (2-3 specific, actionable items)           │
│  - Overall assessment (hire/no-hire if this were real)              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Sample Feedback Template

```
MOCK INTERVIEW FEEDBACK

Candidate: [Name]
Problem: [Problem name]
Date: [Date]

WHAT WENT WELL:
1. [Specific positive behavior]
2. [Specific positive behavior]
3. [Specific positive behavior]

AREAS FOR IMPROVEMENT:
1. [Specific behavior] → [Suggestion]
2. [Specific behavior] → [Suggestion]
3. [Specific behavior] → [Suggestion]

TIME MANAGEMENT:
- Requirements: [X min] (target: 5-7)
- Estimation: [X min] (target: 3-5)
- High-level: [X min] (target: 15-18)
- Deep dive: [X min] (target: 12-15)
- Wrap-up: [X min] (target: 5)

OVERALL ASSESSMENT:
[Hire / Lean Hire / Lean No Hire / No Hire]

REASONING:
[Brief explanation of assessment]
```

### Receiving Feedback Effectively

**Do**:
- Listen without defending
- Take notes
- Ask clarifying questions
- Thank the interviewer
- Reflect on the feedback later

**Don't**:
- Argue with feedback
- Make excuses
- Dismiss feedback you disagree with
- Get discouraged

---

## 6️⃣ Practice Schedule

### Recommended Timeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                    8-WEEK PREPARATION SCHEDULE                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  WEEKS 1-2: FOUNDATION                                              │
│  ─────────────────────                                              │
│  - Study system design fundamentals                                 │
│  - Do 2 self-practice sessions                                      │
│  - Focus on beginner problems                                       │
│                                                                      │
│  WEEKS 3-4: BUILDING SKILLS                                         │
│  ──────────────────────────                                         │
│  - 2 mock interviews per week (peer or Pramp)                       │
│  - Mix of beginner and intermediate problems                        │
│  - Focus on time management                                         │
│                                                                      │
│  WEEKS 5-6: REFINEMENT                                              │
│  ─────────────────────                                              │
│  - 3 mock interviews per week                                       │
│  - Include some paid/professional mocks                             │
│  - Focus on intermediate and advanced problems                      │
│  - Work on weak areas identified in feedback                        │
│                                                                      │
│  WEEKS 7-8: PEAK PERFORMANCE                                        │
│  ───────────────────────────                                        │
│  - 2-3 mock interviews per week                                     │
│  - Focus on problems similar to target company                      │
│  - Simulate real interview conditions                               │
│  - Build confidence                                                 │
│                                                                      │
│  TOTAL: 16-20 mock interviews before real interviews                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Weekly Practice Template

```
WEEKLY PRACTICE TEMPLATE

Monday:
- Study: Review one system design topic (1-2 hours)

Tuesday:
- Mock Interview #1 (1 hour)

Wednesday:
- Study: Review feedback, study weak areas (1-2 hours)

Thursday:
- Mock Interview #2 (1 hour)

Friday:
- Study: Practice problem deep dives (1-2 hours)

Weekend:
- Mock Interview #3 (optional, 1 hour)
- Review week's progress
- Plan next week's focus areas

TOTAL: 6-10 hours per week
```

---

## 7️⃣ Common Mock Interview Mistakes

### Mistake 1: Not Treating It Seriously

**Problem**: Treating mock interviews casually, looking at notes, not managing time.

**Fix**: Treat every mock interview like the real thing. No notes, strict time limits, professional demeanor.

### Mistake 2: Always Being the Candidate

**Problem**: Only practicing as candidate, never as interviewer.

**Fix**: Alternate roles. Being an interviewer teaches you what interviewers look for.

### Mistake 3: Not Incorporating Feedback

**Problem**: Getting feedback but not acting on it.

**Fix**: Keep a feedback log. Track recurring themes. Deliberately practice weak areas.

### Mistake 4: Practicing the Same Problems

**Problem**: Doing the same 5 problems repeatedly.

**Fix**: Vary your problems. Include unfamiliar topics. Practice adapting to new problems.

### Mistake 5: Skipping the Feedback Session

**Problem**: Ending the mock interview without discussing feedback.

**Fix**: Always allocate 10-15 minutes for feedback. It's the most valuable part.

---

## 8️⃣ One Clean Mental Summary

Mock interviews are the most effective way to prepare for system design interviews. Aim for 2-3 mock interviews per week, totaling 16-20 before your real interviews. Use a mix of peer practice (free) and professional platforms (paid) for varied feedback.

Structure each mock interview like the real thing: 45 minutes with strict time management. Always include a feedback session. Track your progress and deliberately practice weak areas.

The goal is to build muscle memory so that structuring your thoughts, managing time, and communicating clearly become automatic. When the real interview comes, you can focus on the problem rather than the process.

