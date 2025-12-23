# 🚀 PHASE 10: MICROSERVICES & ARCHITECTURE (Week 15)

**Goal**: Understand modern software architecture

**Learning Objectives**:
- Design microservices that are independently deployable
- Choose the right communication patterns
- Handle distributed data and transactions
- Apply architectural patterns appropriately

**Estimated Time**: 15-20 hours

---

## Topics:

### Fundamentals

1. **Monolith vs Microservices**
   - When to use each
   - Migration strategies
   - Trade-offs (complexity, latency, consistency)
   - Modular monolith as middle ground

2. **12-Factor App**
   - Codebase, dependencies, config
   - Backing services, build/release/run
   - Processes, port binding, concurrency
   - Disposability, dev/prod parity
   - Logs, admin processes
   - Why it matters for cloud-native

3. **Domain-Driven Design (DDD)**
   - Bounded contexts
   - Aggregates and entities
   - Domain events
   - Context mapping
   - Ubiquitous language
   - Strategic vs tactical DDD

### Service Communication

4. **Service Discovery**
   - Client-side (Eureka, Ribbon)
   - Server-side (Consul, ZooKeeper)
   - DNS-based (Kubernetes)
   - Service registry patterns

5. **REST vs gRPC**
   - When to use each
   - Performance comparison
   - Versioning strategies
   - Error handling differences

6. **Microservices Communication Patterns**
   - Synchronous (REST, gRPC)
   - Asynchronous (messaging)
   - Request/Reply pattern
   - Choreography vs Orchestration
   - Event-driven communication
   - Hybrid approaches

### Data Management

7. **Database per Service**
   - Why separate databases?
   - Data duplication strategies
   - Eventual consistency handling
   - Cross-service queries

8. **CQRS (Command Query Responsibility Segregation)**
   - Separate read/write models
   - Event sourcing integration
   - Use cases and trade-offs
   - Implementation patterns

9. **Saga Pattern**
   - Orchestration vs Choreography
   - Compensating transactions
   - Saga execution coordinator
   - Implementation with events
   - Failure handling

10. **Event-Driven Architecture**
    - Event notification vs Event-carried state transfer
    - Event sourcing
    - CQRS + Event Sourcing
    - Event store design

### Architecture Patterns

11. **Backend for Frontend (BFF)**
    - Why separate backends
    - GraphQL as BFF
    - Mobile vs Web BFF
    - BFF anti-patterns

12. **Strangler Pattern**
    - Incremental migration
    - Monolith to microservices
    - Routing strategies
    - Risk mitigation

13. **Anti-Corruption Layer**
    - Isolating legacy systems
    - Translation between contexts
    - When to use
    - Implementation strategies

14. **API Gateway Patterns**
    - Single entry point
    - Request aggregation
    - Protocol translation
    - Backend for Frontend (BFF) pattern
    - API composition
    - Gateway vs Service Mesh

### Operations

15. **API Versioning**
    - URL versioning (/v1/, /v2/)
    - Header versioning
    - Content negotiation
    - Deprecation strategies

16. **Session Management**
    - Sticky sessions
    - Session replication
    - JWT tokens (stateless)
    - Redis sessions (centralized)
    - Session vs token trade-offs

17. **Service Mesh (Istio/Linkerd)**
    - Sidecar proxy pattern
    - Traffic management
    - Security (mTLS)
    - Observability integration
    - When to adopt service mesh
    - Service mesh vs library approach

18. **Distributed Tracing Integration**
    - OpenTelemetry
    - Trace context propagation
    - Service dependency mapping
    - Performance bottleneck identification
    - Sampling strategies

### Testing & Deployment

19. **Contract Testing**
    - Consumer-driven contracts
    - Pact framework
    - Provider verification
    - Breaking change detection

20. **Microservices Testing Strategies**
    - Unit tests
    - Integration tests
    - Component tests
    - End-to-end tests
    - Test pyramid for microservices

---

## Cross-References:
- **Service Discovery Basics**: See Phase 2
- **Saga Pattern**: Applied in Phase 9 (E-commerce HLD)
- **Event-Driven**: See Phase 6 (Messaging)
- **Service Mesh Basics**: See Phase 5

---

## Practice Problems:
1. Design a migration plan from monolith to microservices
2. Choose between choreography and orchestration for an order flow
3. Design a CQRS implementation for a reporting system
4. Implement service discovery with health checks

---

## Common Interview Questions:
- "When would you NOT use microservices?"
- "How do you handle transactions across services?"
- "How do you debug issues in a microservices architecture?"
- "What's the difference between orchestration and choreography?"

---

## Deliverable
Can architect microservices system for 100+ services
