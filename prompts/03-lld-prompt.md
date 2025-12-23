# 🔹 PROMPT 3: LLD PROMPT (LOW-LEVEL DESIGN)

> **Usage:** Use this for object-oriented design problems.  
> **Examples:** Parking Lot, LRU Cache, ATM, Library Management, Elevator System, etc.

---

```
Design the system: <LLD PROBLEM NAME>

STRICT REQUIREMENTS:

STEP 1: COMPLETE REFERENCE SOLUTION
Present the final, clean design upfront as the "answer key":
- Full class diagram with all classes
- Clear responsibilities for each class
- UML-style relationships (inheritance, composition, aggregation)
- Clean, Java-oriented design using proper OOP
- Show how classes connect and communicate
- This is the target state before explanation begins

STEP 2: DETAILED EXPLANATION
For each class and design decision, explain:
- Why this class exists (what problem it owns)
- Why responsibilities are placed where they are
- How objects interact at runtime
- Which design patterns are used and WHY they fit
- Why alternative designs were rejected
- What would break if this class didn't exist

STEP 3: SOLID PRINCIPLES CHECK
Evaluate the design against each principle:
- Single Responsibility: Does each class do ONE thing?
- Open/Closed: Can we extend without modifying existing code?
- Liskov Substitution: Are subtypes truly interchangeable?
- Interface Segregation: Are interfaces minimal and focused?
- Dependency Inversion: Do we depend on abstractions?
- For any apparent violation, explain why it's acceptable

STEP 4: CODE WALKTHROUGH
Show how engineers write this from scratch:
- What file/class is created first and why
- What comes next in the sequence
- How logic evolves as you build
- How state flows through the system
- Concurrency concerns and thread safety
- Full Java code with comments explaining decisions

STEP 5: SIMULATION / DRY RUN
Walk through concrete examples:
- Show object creation with actual values
- Show method calls in sequence
- Show state changes after each call
- Trace through 2-3 different scenarios
- Include at least one edge case scenario

STEP 6: EDGE CASES & TESTING
- Boundary conditions (empty, null, max values)
- Invalid inputs and error handling
- Concurrent access scenarios
- How would you unit test each class?
- What mocks/stubs are needed?
- Integration test approach

STEP 7: COMPLEXITY ANALYSIS
- Time complexity for each major operation
- Space complexity
- Where are the performance bottlenecks?
- What happens at 10x, 100x usage?

STEP 8: INTERVIEW FOLLOW-UPS (WITH ANSWERS)
Questions and complete answers on:
- How would you scale this?
- How would you handle concurrency?
- How would you extend for new requirements?
- What are the tradeoffs in your design?
- What would you do differently with more time?
- Common interviewer challenges and responses

OUTPUT FILE STRUCTURE:
This prompt generates content for 3 files:
- STEP 1-2, 4-5 → `01-problem-solution.md` (complete code with comments)
- STEP 2-3 → `02-design-explanation.md` (SOLID principles, design decisions, patterns)
- STEP 4-5, 7 → `03-code-walkthrough.md` (line-by-line explanation, simulation, testing)

RULES:
- No skipping fundamentals
- No overengineering beyond requirements
- Explain every design pattern when used
- Prefer clarity over cleverness
- Show the simple version before optimizations
- Time management: Aim for 45-minute solution time
- Level-specific guidance:
  * L4: Focus on working solution, basic OOP
  * L5: SOLID principles, design patterns, edge cases
  * L6: Multiple approaches, advanced patterns, scalability considerations
```

