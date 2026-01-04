Design the system: <LLD PROBLEM NAME>

STRICT REQUIREMENTS:

STEP 0: REQUIREMENTS QUICKPASS
Before designing anything, clearly state:
- Core functional requirements
- Explicit out-of-scope items
- Assumptions and constraints
- Scale assumptions (single process vs multi-threaded, single JVM vs distributed)
- Concurrency model expectations
- Public APIs (main methods that will be exposed)
- Invariants the system must always maintain

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
- Threading model and concurrency control
  * What is shared state
  * Locking strategy or lock-free approach
  * Deadlock and starvation prevention
- Full Java code with comments explaining decisions

STEP 5: SIMULATION / DRY RUN
Walk through concrete examples:
- Show object creation with actual values
- Show method calls in sequence
- Show state changes after each call
- Trace through 2-3 different scenarios
- Include at least one edge case scenario
- Include one concurrency scenario (simultaneous access)

STEP 6: EDGE CASES & TESTING STRATEGY
- Boundary conditions (empty, null, max values)
- Invalid inputs and error handling
- Concurrent access scenarios and race conditions
- Unit test strategy for each class
- Integration test approach
- Load and stress testing approach

STEP 7: COMPLEXITY ANALYSIS
- Time complexity for each major operation
- Space complexity
- Where are the performance bottlenecks?
- What happens at 10x, 100x usage?

STEP 8: INTERVIEW FOLLOW-UPS (WITH ANSWERS)
Questions and complete answers on:
- How would you scale this?
- How would you handle higher concurrency?
- How would you extend for new requirements?
- What are the tradeoffs in your design?
- What would you do differently with more time?
- Common interviewer challenges and responses

OUTPUT FILE STRUCTURE:
This prompt generates content for 3 files:
- STEP 0-1, 4 → `01-problem-solution.md` (final design + full code)
- STEP 2-3, 7 → `02-design-explanation.md` (design decisions, SOLID, complexity)
- STEP 5-6, 8 → `03-simulation-testing.md` (dry runs, edge cases, tests, interview Q&A)

RULES:
- No skipping fundamentals
- No overengineering beyond requirements
- Explain every design pattern when used
- Prefer clarity over cleverness
- Show the simple version before optimizations
- Build baseline first, then list extension points
- Time management: Aim for 45-minute solution time
- Level-specific guidance:
  * L4: Focus on working solution, basic OOP
  * L5: SOLID principles, design patterns, edge cases
  * L6: Multiple approaches, advanced patterns, scalability considerations
