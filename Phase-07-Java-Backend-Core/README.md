# ☕ PHASE 7: JAVA BACKEND CORE (Week 7)

**Goal**: Solid Java fundamentals for backend systems

**Learning Objectives**:
- Master OOP principles and design patterns
- Understand Java concurrency deeply
- Build production-ready Spring Boot applications
- Optimize JVM performance

**Estimated Time**: 20-25 hours

---

## Topics:

1. **OOP Fundamentals**
   - Encapsulation, Inheritance, Polymorphism
   - Composition vs Inheritance (prefer composition)
   - Abstract classes vs Interfaces
   - When to use each

2. **SOLID Principles**
   - Single Responsibility
   - Open/Closed
   - Liskov Substitution
   - Interface Segregation
   - Dependency Inversion
   - Real code examples for each
   - Anti-patterns to avoid

3. **Collections Deep Dive**
   - ArrayList vs LinkedList (when to use)
   - HashMap internals (buckets, load factor, rehashing)
   - TreeMap vs HashMap
   - ConcurrentHashMap (segments, lock striping)
   - Time complexity for all operations
   - Choosing the right collection

4. **Streams API**
   - Functional programming in Java
   - map, filter, reduce, flatMap
   - Collectors (groupingBy, partitioningBy)
   - Parallel streams (when to use, pitfalls)
   - Custom collectors

5. **Exception Handling**
   - Checked vs Unchecked
   - Try-with-resources
   - Custom exceptions
   - Exception handling in REST APIs
   - Global exception handlers

6. **Java Concurrency**
   - Threads (creating, lifecycle)
   - Synchronization (synchronized keyword)
   - Locks (ReentrantLock, ReadWriteLock, StampedLock)
   - ExecutorService & Thread Pools
   - Callable vs Runnable
   - Future & CompletableFuture
   - Atomic types (AtomicInteger, AtomicReference)
   - volatile keyword
   - ThreadLocal

7. **Java Memory Model**
   - Heap vs Stack
   - Happens-before relationship
   - Visibility guarantees
   - Memory leaks (detection and prevention)
   - Weak references, soft references

8. **Garbage Collection**
   - GC basics
   - Generational GC (Young, Old, Metaspace)
   - GC algorithms (Serial, Parallel, CMS, G1, ZGC, Shenandoah)
   - GC tuning basics
   - GC logs analysis

9. **Design Patterns** (Gang of Four)
   - **Creational**: Factory, Abstract Factory, Builder, Singleton, Prototype
   - **Structural**: Adapter, Decorator, Facade, Proxy, Composite
   - **Behavioral**: Strategy, Observer, Template Method, State, Command, Chain of Responsibility
   - Repository pattern (Spring Data)
   - Each with Java code examples

10. **Spring Framework Fundamentals**
    - Dependency Injection (DI)
    - Inversion of Control (IoC)
    - Spring Bean lifecycle
    - Bean scopes (singleton, prototype, request, session)
    - Aspect-Oriented Programming (AOP)
    - Spring Boot auto-configuration
    - Spring MVC architecture

11. **Advanced Java Features**
    - Generics (wildcards, bounded types, type erasure)
    - Lambda expressions & method references
    - Stream API deep dive
    - Optional class (proper usage)
    - Reflection API basics
    - Annotations (custom annotations, meta-annotations)

12. **JVM Tuning & Performance**
    - JVM memory tuning
    - GC tuning parameters
    - JVM flags (-Xmx, -Xms, -XX flags)
    - JIT compilation (C1, C2, tiered)
    - Profiling tools (JProfiler, VisualVM, async-profiler)
    - Flame graphs

13. **Java Best Practices**
    - Effective Java principles
    - Code smells and refactoring
    - Immutability patterns
    - Builder pattern implementation
    - Null safety strategies (Optional, annotations)

14. **Modern Java Features (17+)**
    - Records
    - Sealed classes
    - Pattern matching (instanceof, switch)
    - Text blocks
    - Local variable type inference (var)

15. **Project Loom (Java 21+)**
    - Virtual threads
    - Structured concurrency
    - Why virtual threads matter
    - Migration from thread pools
    - When to use virtual threads

16. **Reactive Programming**
    - Spring WebFlux basics
    - Mono and Flux
    - Backpressure handling
    - When to use reactive
    - Reactive vs imperative trade-offs

17. **Testing in Java**
    - JUnit 5 features
    - Mockito (mocking, stubbing, verification)
    - Integration testing with Spring Boot
    - Test containers
    - TDD workflow

---

## Cross-References:
- **Design Patterns**: Applied in Phase 8 (LLD)
- **Spring Boot**: Used throughout Phase 8-11
- **Concurrency**: Applied in Phase 5 (Distributed Systems)

---

## Practice Problems:
1. Implement a thread-safe LRU cache
2. Build a REST API with proper exception handling
3. Optimize a slow endpoint using profiling
4. Implement the Circuit Breaker pattern from scratch

---

## Common Interview Questions:
- "Explain the difference between synchronized and ReentrantLock"
- "How does HashMap handle collisions?"
- "What's the difference between @Component and @Bean?"
- "How would you debug a memory leak?"

---

## Deliverable
Can explain Spring Boot internals and implement core patterns
