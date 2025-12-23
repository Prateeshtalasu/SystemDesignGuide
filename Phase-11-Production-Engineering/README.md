# ⚙️ PHASE 11: PRODUCTION ENGINEERING (Week 15-16)

**Goal**: Ship, deploy, and operate software

**Learning Objectives**:

- Build and deploy production-ready applications
- Implement observability (logs, metrics, traces)
- Handle incidents and perform root cause analysis
- Design for reliability and disaster recovery

**Estimated Time**: 20-25 hours

---

## Topics:

### Version Control & Collaboration

1. **Git & Collaboration**
   - Branching strategies (Gitflow, trunk-based, GitHub flow)
   - Merge vs Rebase (when to use)
   - Conflict resolution
   - Commit quality (atomic, descriptive)
   - PR best practices
   - Code review guidelines
   - Monorepo vs polyrepo

### Testing

2. **Testing Strategy**
   - Unit tests (JUnit, Mockito)
   - Integration tests
   - Contract tests (Pact)
   - Test pyramid
   - TDD workflows
   - Test coverage (what's enough?)
   - Mutation testing

### Build & Package

3. **Build & Packaging**
   - Maven lifecycle (compile, test, package, install)
   - Gradle basics
   - Dependency management
   - Semantic versioning
   - Artifact repositories (Nexus, Artifactory)
   - Fat JARs vs Thin JARs
   - Build optimization

### Containerization

4. **Docker**
   - Dockerfile (multi-stage builds)
   - Images vs Containers
   - Layers and caching
   - Docker Compose
   - Docker vs VM
   - Local → Prod workflow
   - Docker security best practices
   - Image optimization

### Cloud

5. **Cloud (AWS Focus)**
   - **Compute**: EC2, Lambda, ECS, EKS, Fargate
   - **Storage**: S3, EBS, EFS
   - **Database**: RDS, DynamoDB, ElastiCache, Aurora
   - **Messaging**: SQS, SNS, Kinesis, EventBridge
   - **Networking**: VPC, ELB, API Gateway, CloudFront, Route53
   - **IAM**: Roles, policies, least privilege
   - **Cost optimization basics**

### Orchestration

6. **Kubernetes**
   - Pods (lifecycle, multi-container)
   - Deployments (rolling updates, rollbacks)
   - Services (ClusterIP, NodePort, LoadBalancer)
   - Ingress (routing, TLS)
   - ConfigMaps & Secrets
   - Horizontal Pod Autoscaler (HPA)
   - Vertical Pod Autoscaler (VPA)
   - Readiness & Liveness probes
   - Resource limits and requests
   - Namespaces and RBAC

### CI/CD

7. **CI/CD**

   - Jenkins (pipelines, Jenkinsfile)
   - GitHub Actions (workflows, runners)
   - GitLab CI
   - Build → Test → Deploy pipeline
   - Blue-Green deployment
   - Canary deployment
   - Rolling deployment
   - Feature flags integration

8. **Feature Flags**
   - Why feature flags?
   - LaunchDarkly, Unleash, Flagsmith
   - Flag types (release, experiment, ops)
   - Flag lifecycle management
   - Kill switches
   - Gradual rollouts

### Infrastructure as Code

9. **Infrastructure as Code**
   - Terraform basics
   - State management
   - Modules
   - Rollback strategies
   - Terraform vs CloudFormation
   - GitOps principles

### Observability

10. **Observability**

    - **Logging**: Structured logs (JSON), log levels, correlation IDs
    - **Metrics**: RED (Rate, Errors, Duration), USE (Utilization, Saturation, Errors)
    - **Tracing**: Distributed tracing (Jaeger, Zipkin)
    - Prometheus + Grafana
    - ELK Stack (Elasticsearch, Logstash, Kibana)
    - Alerting (PagerDuty, OpsGenie)
    - Dashboards best practices

11. **Monitoring vs Observability**
    - Three pillars (logs, metrics, traces)
    - Observability maturity model
    - SLI/SLO/SLA deep dive
    - Error budgets
    - Monitoring best practices
    - Cardinality management

### Incident Management

12. **Debugging & On-Call**

    - Reading stack traces
    - Thread dumps, heap dumps
    - Root Cause Analysis (5 Whys)
    - Incident response (runbooks)
    - Postmortems (blameless)
    - On-call rotations
    - Escalation procedures

13. **On-Call Best Practices**
    - Runbook creation
    - Alert fatigue prevention
    - Escalation policies
    - Handoff procedures
    - Documentation standards
    - Incident severity levels

### Resilience

14. **Chaos Engineering**

    - Failure injection
    - Chaos experiments
    - Tools (Chaos Monkey, Chaos Mesh, Litmus)
    - Game days
    - Building resilience
    - Steady state hypothesis

15. **Load Testing & Performance**

    - Load testing tools (JMeter, Gatling, k6)
    - Stress testing
    - Capacity planning
    - Performance benchmarking
    - Bottleneck identification
    - Performance budgets

16. **Disaster Recovery**

    - Backup strategies (full, incremental, differential)
    - Recovery Time Objective (RTO)
    - Recovery Point Objective (RPO)
    - Multi-region deployment
    - Failover strategies (active-passive, active-active)
    - DR testing

17. **Capacity Planning**
    - Forecasting traffic growth
    - Resource provisioning
    - Auto-scaling strategies
    - Cost vs performance trade-offs
    - Capacity reviews
    - _Reference_: See Phase 1 (Back-of-envelope calculations)

---

## Cross-References:

- **Back-of-Envelope**: See Phase 1, Topic 11
- **Chaos Engineering Intro**: See Phase 1, Topic 10
- **Distributed Tracing**: See Phase 5

---

## Practice Problems:

1. Set up a CI/CD pipeline with blue-green deployment
2. Create a Kubernetes deployment with HPA
3. Design a monitoring dashboard for a microservice
4. Write a runbook for a database failover

---

## Common Interview Questions:

- "How would you debug a production issue?"
- "What's in your CI/CD pipeline?"
- "How do you handle a sudden traffic spike?"
- "What metrics would you monitor for a payment service?"

---

## Deliverable

Can deploy and monitor production system
