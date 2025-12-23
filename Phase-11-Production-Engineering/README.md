# ⚙️ PHASE 11: PRODUCTION ENGINEERING (Week 15-16)

**Goal**: Ship, deploy, and operate software

## Topics:

1. **Git & Collaboration**

   - Branching strategies (Gitflow, trunk-based)
   - Merge vs Rebase (when to use)
   - Conflict resolution
   - Commit quality (atomic, descriptive)
   - PR best practices
   - Code review guidelines

2. **Testing Strategy**

   - Unit tests (JUnit, Mockito)
   - Integration tests
   - Contract tests (Pact)
   - Test pyramid
   - TDD workflows
   - Test coverage (what's enough?)

3. **Build & Packaging**

   - Maven lifecycle (compile, test, package, install)
   - Dependency management
   - Semantic versioning
   - Artifact repositories (Nexus, Artifactory)
   - Fat JARs vs Thin JARs

4. **Docker**

   - Dockerfile (multi-stage builds)
   - Images vs Containers
   - Layers and caching
   - Docker Compose
   - Docker vs VM
   - Local → Prod workflow

5. **Cloud (AWS Focus)**

   - **Compute**: EC2, Lambda, ECS, EKS
   - **Storage**: S3, EBS, EFS
   - **Database**: RDS, DynamoDB, ElastiCache
   - **Messaging**: SQS, SNS, Kinesis
   - **Networking**: VPC, ELB, API Gateway, CloudFront
   - **IAM**: Roles, policies, least privilege

6. **Kubernetes**

   - Pods (lifecycle)
   - Deployments (rolling updates, rollbacks)
   - Services (ClusterIP, NodePort, LoadBalancer)
   - Ingress (routing)
   - ConfigMaps & Secrets
   - Horizontal Pod Autoscaler (HPA)
   - Readiness & Liveness probes

7. **CI/CD**

   - Jenkins (pipelines, Jenkinsfile)
   - GitHub Actions (workflows, runners)
   - Build → Test → Deploy pipeline
   - Blue-Green deployment
   - Canary deployment
   - Feature flags

8. **Infrastructure as Code**

   - Terraform basics
   - State management
   - Modules
   - Rollback strategies

9. **Observability**

   - **Logging**: Structured logs (JSON), log levels, correlation IDs
   - **Metrics**: RED (Rate, Errors, Duration), USE (Utilization, Saturation, Errors)
   - **Tracing**: Distributed tracing (Jaeger, Zipkin)
   - Prometheus + Grafana
   - Alerting (PagerDuty)

10. **Debugging & On-Call**

    - Reading stack traces
    - Thread dumps, heap dumps
    - Root Cause Analysis (5 Whys)
    - Incident response (runbooks)
    - Postmortems (blameless)

11. **Monitoring vs Observability**

    - Three pillars (logs, metrics, traces)
    - Observability maturity model
    - SLI/SLO/SLA deep dive
    - Error budgets
    - Monitoring best practices

12. **Chaos Engineering**

    - Failure injection
    - Chaos experiments
    - Tools (Chaos Monkey, Chaos Mesh)
    - Game days
    - Building resilience

13. **Load Testing & Performance**

    - Load testing tools (JMeter, Gatling, k6)
    - Stress testing
    - Capacity planning
    - Performance benchmarking
    - Bottleneck identification

14. **Disaster Recovery**
    - Backup strategies
    - Recovery Time Objective (RTO)
    - Recovery Point Objective (RPO)
    - Multi-region deployment
    - Failover strategies

## Deliverable

Can deploy and monitor production system
