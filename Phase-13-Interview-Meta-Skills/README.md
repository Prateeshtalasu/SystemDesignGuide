# 🧠 PHASE 13: INTERVIEW META-SKILLS (Week 16)

**Goal**: Master the interview game

**Learning Objectives**:
- Structure system design interviews effectively
- Make and defend trade-off decisions
- Communicate clearly under pressure
- Handle curveballs gracefully

**Estimated Time**: 10-15 hours

---

## Topics:

### Interview Structure

1. **Problem Approach Framework**
   - Clarify requirements (5-10 min)
   - High-level design (15-20 min)
   - Deep dive (15-20 min)
   - Trade-offs & alternatives (5-10 min)
   - Wrap-up & questions (5 min)

2. **Requirements Clarification**
   - Functional requirements
   - Non-functional (scale, latency, consistency)
   - Out of scope
   - Template questions to ask
   - Prioritization

3. **Back-of-Envelope Calculations**
   - QPS calculation
   - Storage estimation
   - Bandwidth estimation
   - Memory (cache) sizing
   - Practice 20+ examples
   - *Reference*: See Phase 1, Topic 11

### Communication

4. **Communication Tips**
   - Think out loud
   - Draw diagrams
   - Ask for feedback ("Does this make sense?")
   - Handle pushback gracefully
   - Admit uncertainty appropriately

5. **Time Management**
   - 45-min interview breakdown
   - Don't get stuck on one component
   - Breadth before depth
   - When to move on
   - Clock awareness

6. **Drawing & Communication**
   - Whiteboard best practices
   - Diagram types (sequence, component, deployment)
   - Using standard symbols
   - Explaining complex concepts simply
   - Virtual whiteboard tools

7. **Handling "I Don't Know"**
   - When to admit uncertainty
   - How to phrase it
   - Pivoting to related knowledge
   - Asking clarifying questions
   - Thinking out loud through unknowns

### Design Patterns

8. **Common System Design Patterns**
   - Fan-out on write vs Fan-out on read
   - Write-through vs Write-behind caching
   - Sharding strategies
   - Replication patterns
   - CQRS pattern
   - Event-driven architecture
   - Saga pattern

9. **Trade-off Analysis Framework**
   - Consistency vs Availability
   - Latency vs Throughput
   - Cost vs Performance
   - Complexity vs Scalability
   - How to articulate trade-offs
   - "It depends" with substance

10. **Scalability Patterns**
    - Horizontal vs Vertical scaling
    - Database scaling patterns
    - Cache scaling patterns
    - Message queue scaling
    - CDN for static content
    - Microservices decomposition

### Interview Scenarios

11. **Common Interview Scenarios**
    - "Design for 1M users, then scale to 1B"
    - "How would you handle 10x traffic spike?"
    - "What if this component fails?"
    - "How do you ensure data consistency?"
    - "What are the bottlenecks?"
    - "How would you monitor this?"

12. **Common Follow-ups by Topic**
    - Database: "How do you handle hot partitions?"
    - Cache: "What if cache goes down?"
    - Messaging: "How do you ensure ordering?"
    - Consistency: "What if network partitions?"
    - Scale: "How do you handle 10x growth?"
    - Learn 50+ follow-ups

13. **Level-Specific Expectations**
    - **L4 (New Grad)**: Basic design, knows trade-offs
    - **L5 (Mid-level)**: End-to-end design, deep dives
    - **L6 (Senior)**: Multiple solutions, complex trade-offs
    - **L7 (Staff)**: System of systems, org-level impact
    - Calibrate your answers

### Practice & Preparation

14. **Mock Interview Structure**
    - Finding practice partners
    - Pramp, Interviewing.io
    - Self-recording and review
    - Feedback incorporation
    - Frequency (2-3 per week)

15. **Common Mistakes to Avoid**
    - Jumping to solution too fast
    - Not clarifying requirements
    - Ignoring non-functional requirements
    - Over-engineering
    - Under-engineering
    - Not discussing trade-offs
    - Poor time management
    - Not asking questions

16. **Day-of Preparation**
    - Sleep and rest
    - Environment setup
    - Materials ready
    - Mental preparation
    - Warm-up routine

17. **Post-Interview Analysis**
    - Self-evaluation
    - What went well
    - What to improve
    - Updating notes
    - Continuous improvement

---

## Quick Reference: Interview Checklist

### Before Design
- [ ] Clarify functional requirements
- [ ] Clarify non-functional requirements (scale, latency)
- [ ] Identify out of scope
- [ ] Do capacity estimation

### During Design
- [ ] Start with high-level architecture
- [ ] Explain each component's purpose
- [ ] Draw data flow
- [ ] Discuss database schema
- [ ] Address scalability
- [ ] Handle failure scenarios

### Deep Dive
- [ ] Pick 1-2 interesting components
- [ ] Explain internals
- [ ] Discuss alternatives
- [ ] Justify choices

### Wrap-up
- [ ] Summarize design
- [ ] Discuss monitoring
- [ ] Mention future improvements
- [ ] Ask questions

---

## Practice Problems:
1. Complete 10 mock interviews with timer
2. Record yourself designing a system
3. Practice explaining trade-offs out loud
4. Create a personal cheat sheet

---

## Common Interview Questions:
- "Walk me through your approach"
- "What would you do differently?"
- "How would you prioritize features?"
- "What questions do you have for me?"

---

## Resources:
- System Design Primer (GitHub)
- Designing Data-Intensive Applications (book)
- Engineering blogs (Netflix, Uber, Airbnb)
- YouTube channels (System Design Interview)

---

## Deliverable
Can ace 10 mock interviews in a row
