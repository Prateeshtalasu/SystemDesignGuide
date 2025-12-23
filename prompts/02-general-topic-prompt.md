# 🔹 PROMPT 2: GENERAL TOPIC PROMPT (CONCEPTS / TOOLS / THEORY)

> **Usage:** Use this for any single concept.  
> **Examples:** Redis, Kafka, DNS, JWT, Docker, Circuit Breaker, Load Balancer, etc.

---

```
Explain the topic: <TOPIC NAME>

Follow this exact structure:

0️⃣ Prerequisites
   - What concepts must be understood before this topic
   - Quick 1-2 sentence refresher for simple prerequisites
   - Explicit reference to prior topic if complex

1️⃣ What problem does this exist to solve?
   - The specific pain point that created the need
   - What systems looked like before this existed
   - What breaks, fails, or becomes painful without it
   - Real examples of the problem

2️⃣ Intuition and mental model
   - Use a concrete analogy from everyday life
   - Build a mental picture that sticks
   - This analogy will be referenced throughout

3️⃣ How it works internally
   - Step-by-step mechanics
   - Lifecycle from start to finish
   - Internal data structures and algorithms
   - What happens at each stage

4️⃣ Simulation-first explanation
   - Start with the smallest possible example
   - 1 user, 1 service, 1 DB/cache/broker
   - Walk through exact data movement
   - Show what bytes/messages/requests look like
   - Trace a single request end-to-end

5️⃣ How engineers actually use this in production
   - Real systems at real companies (name them: Uber, Netflix, etc.)
   - Real workflows and tooling
   - What is automated vs manually configured
   - Link to engineering blogs or talks if notable
   - Production war stories and lessons learned

6️⃣ How to implement or apply it (apply this wherever it required)
   - Code in Java (idiomatic, production-style)
   - Maven/Gradle dependencies if applicable
   - Spring Boot integration if relevant
   - Config files (application.yml, docker-compose, etc.)
   - Commands to run and verify
   - How engineers write this from scratch, step by step

7️⃣ Tradeoffs, pitfalls, and common mistakes
   - Why not alternative X?
   - What breaks at scale?
   - Common misconfigurations
   - Performance gotchas
   - Security considerations

8️⃣ When NOT to use this
   - Anti-patterns and misuse cases
   - Situations where this is overkill
   - Better alternatives for specific scenarios
   - Signs you've chosen the wrong tool

9️⃣ Comparison with Alternatives
   - Why this over alternative X?
   - When to choose this vs alternatives
   - Migration path from alternatives
   - Trade-offs vs other solutions
   - Real-world examples of companies choosing this

🔟 Interview follow-up questions WITH answers
   - Questions an interviewer would ask
   - Complete answers, not just hints
   - Common variations and curveballs
   - Level-specific questions (L4 vs L5 vs L6)

1️⃣1️⃣ One clean mental summary
   - 3-5 sentences that capture the essence
   - Something you could explain in an elevator

RULES:
- Explain every new term before using it
- Do not rush through any section
- Avoid surface-level summaries
- If a section doesn't apply, explicitly state why and skip it
```

