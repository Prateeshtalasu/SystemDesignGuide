# 🔄 CI/CD

## 0️⃣ Prerequisites

Before diving into CI/CD, you should understand:

- **Git**: Version control, branches, commits (covered in Topic 1)
- **Build Tools**: Maven/Gradle, how to build and package applications (covered in Topic 3)
- **Docker**: Containerization basics (covered in Topic 4)
- **Cloud/Kubernetes**: Deployment targets (covered in Topics 5-6)

Quick refresher on **automation**: Automation means using scripts and tools to perform tasks without manual intervention. CI/CD automates the process of building, testing, and deploying software.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Before CI/CD

Imagine deploying a Java application manually:

**Problem 1: The "Works on My Machine" Deployment**

```
Developer's laptop:
1. git pull origin main
2. mvn clean package
3. Tests pass locally
4. Copy JAR to production server via SCP
5. SSH into server
6. Stop old application
7. Start new application
8. Check logs
9. Hope it works
```

What could go wrong:

- Developer forgot to pull latest changes
- Tests pass locally but fail in production environment
- Wrong JAR file copied
- Application doesn't start (missing environment variable)
- No rollback plan if it breaks

**Problem 2: The Friday 5 PM Deployment**

```
Friday 5:00 PM: Developer deploys new version
Friday 5:15 PM: Production is down
Friday 5:30 PM: Can't rollback (forgot to backup old version)
Friday 6:00 PM: Emergency call to entire team
Saturday 2:00 AM: Finally fixed, everyone exhausted
```

Manual deployments are risky, especially on Fridays.

**Problem 3: Inconsistent Environments**

```
Development: Java 17, PostgreSQL 15, Redis 7
Staging: Java 11, PostgreSQL 14, Redis 6
Production: Java 17, PostgreSQL 15, Redis 7

Code works in dev, fails in staging, works in production.
Why? Different versions, different configurations.
```

**Problem 4: Slow Feedback Loops**

Developer commits code on Monday.

- Code review happens on Tuesday
- Manual testing on Wednesday
- Deployment to staging on Thursday
- Production deployment on Friday

Bug introduced Monday isn't discovered until Friday. Context is lost, fixing is harder.

**Problem 5: Human Error**

```
Manual deployment checklist:
□ Pull latest code
□ Run tests
□ Build JAR
□ Backup database
□ Stop application
□ Deploy new JAR
□ Start application
□ Verify health endpoint
□ Check logs
□ Update deployment log

Step 7: Developer accidentally runs "rm -rf /app/*" instead of deploying
Result: Entire application deleted, 4 hours to restore from backup
```

### What Breaks Without CI/CD

| Scenario             | Without CI/CD             | With CI/CD                   |
| -------------------- | ------------------------- | ---------------------------- |
| Deployment frequency | Weekly or monthly         | Multiple times per day       |
| Time to deploy       | Hours                     | Minutes                      |
| Deployment errors    | Common (human error)      | Rare (automated)             |
| Rollback time        | Hours (manual)            | Seconds (automated)          |
| Feedback on bugs     | Days later                | Minutes after commit         |
| Consistency          | Different per environment | Identical process everywhere |

---

## 2️⃣ Intuition and Mental Model

### The Assembly Line Analogy

Think of CI/CD as an **automated assembly line** for software.

**Before assembly lines** (craftsman model):

- One person builds entire car
- Takes weeks
- Quality varies
- Expensive
- Hard to scale

**With assembly line**:

- Each station does one thing well
- Fast and consistent
- Quality checks at each stage
- If something fails, line stops
- Easy to scale

CI/CD is the same:

- **Source**: Code repository
- **Station 1**: Build (compile code)
- **Station 2**: Test (run tests)
- **Station 3**: Package (create artifact)
- **Station 4**: Deploy (ship to environment)
- **Quality gates**: If any stage fails, pipeline stops

### CI vs CD

**CI (Continuous Integration)**:

- Developers integrate code frequently
- Automated build and test on every commit
- Catches integration issues early
- "Does my code work with everyone else's?"

**CD (Continuous Delivery)**:

- Code is always deployable
- Automated deployment to staging
- Manual approval for production
- "Can I deploy this right now?"

**CD (Continuous Deployment)**:

- Every change that passes tests goes to production
- No manual approval
- "Is this safe to deploy automatically?"

```
┌─────────────────────────────────────────────────────────────────┐
│                    CI/CD PIPELINE                                │
│                                                                  │
│  Developer commits code                                         │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              CI (Continuous Integration)                   │   │
│  │                                                           │   │
│  │  1. Build (compile, package)                            │   │
│  │  2. Unit tests                                           │   │
│  │  3. Integration tests                                     │   │
│  │  4. Code quality checks (linting, security)              │   │
│  │                                                           │   │
│  │  If any step fails → Stop, notify developer             │   │
│  └──────────────────────────────────────────────────────────┘   │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │         CD (Continuous Delivery/Deployment)                │   │
│  │                                                           │   │
│  │  5. Deploy to staging                                    │   │
│  │  6. Run E2E tests                                         │   │
│  │  7. Manual approval (if Continuous Delivery)             │   │
│  │  8. Deploy to production                                 │   │
│  │  9. Smoke tests                                          │   │
│  │                                                           │   │
│  │  If any step fails → Rollback automatically              │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ How It Works Internally

### Pipeline Execution Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    PIPELINE EXECUTION                            │
│                                                                  │
│  1. Trigger Event                                               │
│     - Push to branch                                            │
│     - Pull request opened                                       │
│     - Scheduled (cron)                                          │
│     - Manual trigger                                            │
│                                                                  │
│  2. Pipeline Definition Loaded                                   │
│     - Read from repository (Jenkinsfile, .github/workflows)      │
│     - Parse YAML/DSL                                            │
│     - Validate syntax                                           │
│                                                                  │
│  3. Agent/Runner Provisioned                                     │
│     - Allocate compute resource                                 │
│     - Install dependencies                                      │
│     - Clone repository                                          │
│                                                                  │
│  4. Stages Execute Sequentially                                 │
│     Stage 1: Build                                              │
│       - Run commands                                            │
│       - Capture output                                          │
│       - Check exit code                                         │
│                                                                  │
│     Stage 2: Test                                                │
│       - Run test suite                                          │
│       - Collect test results                                    │
│       - Generate coverage report                                │
│                                                                  │
│     Stage 3: Deploy                                             │
│       - Build Docker image                                       │
│       - Push to registry                                        │
│       - Update Kubernetes deployment                            │
│                                                                  │
│  5. Artifacts Stored                                             │
│     - Build artifacts (JARs, Docker images)                    │
│     - Test reports                                              │
│     - Logs                                                      │
│                                                                  │
│  6. Notifications Sent                                           │
│     - Success: Slack, email                                      │
│     - Failure: PagerDuty, Slack                                 │
│                                                                  │
│  7. Cleanup                                                      │
│     - Remove temporary files                                    │
│     - Release agent/runner                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Pipeline State Management

```
Pipeline State Machine:

    START
      │
      ▼
   QUEUED ──────▶ RUNNING ──────▶ SUCCESS
      │              │              │
      │              │              │
      │              ▼              │
      │          FAILED ─────────────┘
      │              │
      │              ▼
      └───────── CANCELLED
```

**State transitions**:

- **QUEUED**: Waiting for available runner
- **RUNNING**: Currently executing stages
- **SUCCESS**: All stages passed
- **FAILED**: Stage failed, pipeline stopped
- **CANCELLED**: Manually cancelled or timeout

### Artifact Management

```
┌─────────────────────────────────────────────────────────────────┐
│                    ARTIFACT LIFECYCLE                            │
│                                                                  │
│  Build Stage                                                    │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────┐                                              │
│  │  JAR file     │  Created during build                       │
│  │  (myapp.jar)  │                                              │
│  └──────────────┘                                              │
│       │                                                         │
│       │ Uploaded to artifact store                             │
│       ▼                                                         │
│  ┌──────────────┐                                              │
│  │ Artifact     │  Stored with metadata:                       │
│  │ Repository   │  - Build number                              │
│  │              │  - Git commit SHA                            │
│  │              │  - Timestamp                                 │
│  │              │  - Build environment                         │
│  └──────────────┘                                              │
│       │                                                         │
│       │ Downloaded in deploy stage                              │
│       ▼                                                         │
│  ┌──────────────┐                                              │
│  │  Deployment  │  Same artifact used in all environments     │
│  │  Target      │                                              │
│  └──────────────┘                                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Simulation: Complete CI/CD Pipeline

### Step 1: Simple GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

# When to trigger
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      # Step 1: Checkout code
      - name: Checkout code
        uses: actions/checkout@v4

      # Step 2: Set up JDK
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: "17"
          distribution: "temurin"
          cache: maven

      # Step 3: Build
      - name: Build with Maven
        run: mvn clean package -DskipTests

      # Step 4: Run tests
      - name: Run tests
        run: mvn test

      # Step 5: Upload test results
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: target/surefire-reports/

      # Step 6: Upload JAR artifact
      - name: Upload JAR
        uses: actions/upload-artifact@v4
        with:
          name: application-jar
          path: target/*.jar
```

### Step 2: Add Docker Build

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: "17"
          distribution: "temurin"
          cache: maven

      - name: Build with Maven
        run: mvn clean package -DskipTests

      - name: Run tests
        run: mvn test

      - name: Generate coverage report
        run: mvn jacoco:report

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./target/site/jacoco/jacoco.xml

  build-docker:
    needs: build-and-test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Step 3: Add Deployment

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  workflow_run:
    workflows: ["CI/CD Pipeline"]
    types:
      - completed
    branches: [main]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: "latest"

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig --name my-cluster --region us-east-1

      - name: Deploy to Kubernetes
        run: |
          # Update image tag in deployment
          sed -i "s|image:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}|" k8s/deployment.yaml

          # Apply deployment
          kubectl apply -f k8s/deployment.yaml

          # Wait for rollout
          kubectl rollout status deployment/myapp

      - name: Run smoke tests
        run: |
          # Wait for service to be ready
          sleep 30

          # Check health endpoint
          HEALTH_URL=$(kubectl get service myapp -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
          curl -f https://$HEALTH_URL/actuator/health || exit 1
```

---

## 5️⃣ Jenkins Deep Dive

### Jenkins Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        JENKINS SERVER                            │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Jenkins Controller                     │   │
│  │                                                           │   │
│  │  - Schedules builds                                        │   │
│  │  - Manages pipelines                                      │   │
│  │  - Stores configuration                                   │   │
│  │  - Serves web UI                                         │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              │ Dispatches jobs                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Jenkins Agents                          │   │
│  │                    (Build Runners)                        │   │
│  │                                                           │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │   Agent 1    │  │   Agent 2    │  │   Agent 3    │   │   │
│  │  │  (Linux)     │  │  (Windows)   │  │  (Docker)    │   │   │
│  │  │              │  │              │  │              │   │   │
│  │  │  Executes    │  │  Executes    │  │  Executes    │   │   │
│  │  │  build steps │  │  build steps │  │  build steps │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Jenkinsfile (Declarative Pipeline)

```groovy
// Jenkinsfile
pipeline {
    agent any  // Run on any available agent

    environment {
        // Environment variables available to all stages
        MAVEN_HOME = tool 'Maven-3.9'
        JAVA_HOME = tool 'JDK-17'
        DOCKER_REGISTRY = 'registry.example.com'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    tools {
        maven 'Maven-3.9'
        jdk 'JDK-17'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                    publishHTML([
                        reportDir: 'target/site/jacoco',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    def image = docker.build("${DOCKER_REGISTRY}/myapp:${IMAGE_TAG}")
                    image.push()
                    image.push("latest")
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                sh 'kubectl set image deployment/myapp myapp=${DOCKER_REGISTRY}/myapp:${IMAGE_TAG} -n staging'
                sh 'kubectl rollout status deployment/myapp -n staging'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                sh 'kubectl set image deployment/myapp myapp=${DOCKER_REGISTRY}/myapp:${IMAGE_TAG} -n production'
                sh 'kubectl rollout status deployment/myapp -n production'
            }
        }
    }

    post {
        always {
            cleanWs()  // Clean workspace
        }
        success {
            emailext(
                subject: "Build Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build completed successfully.",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
        failure {
            emailext(
                subject: "Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build failed. Check console output.",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
    }
}
```

### Jenkinsfile (Scripted Pipeline)

```groovy
// Jenkinsfile (Scripted)
node {
    stage('Checkout') {
        checkout scm
    }

    stage('Build') {
        def mvnHome = tool 'Maven-3.9'
        env.PATH = "${mvnHome}/bin:${env.PATH}"
        sh 'mvn clean package -DskipTests'
    }

    stage('Test') {
        sh 'mvn test'
        junit 'target/surefire-reports/*.xml'
    }

    stage('Deploy') {
        if (env.BRANCH_NAME == 'main') {
            sh 'kubectl apply -f k8s/'
        }
    }
}
```

---

## 6️⃣ Deployment Strategies

### Blue-Green Deployment

```
┌─────────────────────────────────────────────────────────────────┐
│                    BLUE-GREEN DEPLOYMENT                         │
│                                                                  │
│  Before Deployment:                                             │
│                                                                  │
│  ┌──────────────┐                                               │
│  │ Load        │                                               │
│  │ Balancer    │ ──────▶ ┌──────────────┐                      │
│  └──────────────┘        │   Blue       │                      │
│                          │  (v1.0.0)    │                      │
│                          │  3 instances │                      │
│                          └──────────────┘                      │
│                                                                  │
│                          ┌──────────────┐                      │
│                          │   Green      │                      │
│                          │  (v2.0.0)    │                      │
│                          │  0 instances │                      │
│                          └──────────────┘                      │
│                                                                  │
│  During Deployment:                                             │
│                                                                  │
│  ┌──────────────┐                                               │
│  │ Load        │ ──────▶ ┌──────────────┐                      │
│  │ Balancer    │        │   Blue       │                      │
│  └──────────────┘        │  (v1.0.0)    │                      │
│                          │  3 instances │                      │
│                          └──────────────┘                      │
│                                                                  │
│                          ┌──────────────┐                      │
│                          │   Green      │                      │
│                          │  (v2.0.0)    │                      │
│                          │  3 instances │ ← Deploy new version │
│                          └──────────────┘                      │
│                                                                  │
│  After Switch:                                                  │
│                                                                  │
│  ┌──────────────┐                                               │
│  │ Load        │ ──────▶ ┌──────────────┐                      │
│  │ Balancer    │        │   Green      │                      │
│  └──────────────┘        │  (v2.0.0)    │                      │
│                          │  3 instances │                      │
│                          └──────────────┘                      │
│                                                                  │
│                          ┌──────────────┐                      │
│                          │   Blue       │                      │
│                          │  (v1.0.0)    │                      │
│                          │  0 instances │ ← Keep for rollback  │
│                          └──────────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

**Implementation in CI/CD**:

```yaml
# GitHub Actions workflow
- name: Blue-Green Deployment
  run: |
    # Deploy green environment
    kubectl apply -f k8s/green-deployment.yaml

    # Wait for green to be ready
    kubectl rollout status deployment/myapp-green

    # Run smoke tests on green
    ./scripts/smoke-tests.sh green

    # Switch traffic to green
    kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"green"}}}'

    # Monitor for 5 minutes
    sleep 300

    # If healthy, delete blue. If not, rollback
    if ./scripts/health-check.sh; then
      kubectl delete deployment myapp-blue
    else
      kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"blue"}}}'
      kubectl delete deployment myapp-green
    fi
```

### Canary Deployment

```
┌─────────────────────────────────────────────────────────────────┐
│                      CANARY DEPLOYMENT                           │
│                                                                  │
│  Step 1: Deploy 10% traffic to new version                    │
│                                                                  │
│  ┌──────────────┐                                               │
│  │ Load        │ ──── 90% ────▶ ┌──────────────┐              │
│  │ Balancer    │                │   Stable     │              │
│  └──────────────┘                │  (v1.0.0)    │              │
│         │                        │  27 pods    │              │
│         │ 10%                    └──────────────┘              │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │   Canary     │                                               │
│  │  (v2.0.0)    │                                               │
│  │  3 pods      │                                               │
│  └──────────────┘                                               │
│                                                                  │
│  Step 2: Monitor metrics (5 minutes)                            │
│  - Error rate: OK                                               │
│  - Latency: OK                                                  │
│  - CPU/Memory: OK                                               │
│                                                                  │
│  Step 3: Increase to 50%                                         │
│  Step 4: Monitor (5 minutes)                                    │
│  Step 5: Increase to 100%                                        │
│  Step 6: Remove old version                                      │
└─────────────────────────────────────────────────────────────────┘
```

**Implementation with Istio**:

```yaml
# VirtualService for canary
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp.example.com
  http:
    - match:
        - headers:
            canary:
              exact: "true"
      route:
        - destination:
            host: myapp
            subset: v2
          weight: 100
    - route:
        - destination:
            host: myapp
            subset: v1
          weight: 90
        - destination:
            host: myapp
            subset: v2
          weight: 10
```

### Rolling Deployment (Kubernetes Default)

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2 # Can have 2 extra pods during update
      maxUnavailable: 1 # Can have 1 pod unavailable
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:2.0.0
```

**Rolling update process**:

```
Time 0: [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1]
Time 1: [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v2] [v2]
Time 2: [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v2] [v2] [v2]
Time 3: [v1] [v1] [v1] [v1] [v1] [v1] [v2] [v2] [v2] [v2] [v2]
...
Time N: [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2]
```

---

## 7️⃣ Feature Flags Integration

### Feature Flags in CI/CD

```java
// Feature flag service
@Service
public class FeatureFlagService {

    @Autowired
    private FeatureFlagClient flagClient;  // LaunchDarkly, Unleash, etc.

    public boolean isFeatureEnabled(String flagKey, String userId) {
        return flagClient.isEnabled(flagKey, userId);
    }
}
```

```java
// Using feature flag in code
@RestController
public class PaymentController {

    @Autowired
    private FeatureFlagService featureFlag;

    @PostMapping("/payments")
    public PaymentResponse processPayment(@RequestBody PaymentRequest request) {
        // New payment processor behind feature flag
        if (featureFlag.isFeatureEnabled("new-payment-processor", request.getUserId())) {
            return newPaymentProcessor.process(request);
        } else {
            return oldPaymentProcessor.process(request);
        }
    }
}
```

**CI/CD workflow with feature flags**:

```yaml
# Deploy code with feature flag disabled
- name: Deploy with feature flag off
  run: |
    kubectl set image deployment/myapp myapp=myapp:2.0.0
    # Feature flag "new-payment-processor" = false (default)

- name: Enable feature flag for 1% of users
  run: |
    # Via feature flag service API
    curl -X POST https://flags.example.com/flags/new-payment-processor \
      -d '{"percentage": 1}'

- name: Monitor metrics
  run: |
    # Wait and check error rates, latency

- name: Gradually increase rollout
  run: |
    # Increase to 10%, then 50%, then 100%
    # Rollback by disabling flag if issues detected
```

---

## 8️⃣ Tradeoffs and Common Mistakes

### CI/CD Anti-Patterns

**1. Flaky Tests in Pipeline**

```yaml
# BAD: Tests that sometimes pass, sometimes fail
- name: Run tests
  run: mvn test # Flaky test causes pipeline to fail randomly
```

**Solution**: Fix flaky tests, or quarantine them in separate pipeline.

**2. Long-Running Pipelines**

```
Pipeline takes 2 hours:
- Build: 10 minutes
- Tests: 90 minutes
- Deploy: 20 minutes

Developer waits 2 hours for feedback.
```

**Solution**: Parallelize tests, use test sharding, optimize build times.

**3. Secrets in Code**

```yaml
# BAD: Hardcoded secrets
env:
  DATABASE_PASSWORD: "supersecret123"
```

**Solution**: Use secret management (AWS Secrets Manager, HashiCorp Vault, CI/CD secrets).

**4. No Rollback Strategy**

```yaml
# BAD: Deploy and hope
- name: Deploy
  run: kubectl apply -f k8s/
  # No rollback if it fails
```

**Solution**: Always have automated rollback.

**5. Deploying on Fridays**

**Solution**: Use deployment windows, feature flags to deploy safely.

### CI/CD Tool Comparison

| Tool           | Type          | Best For                        | Pros                                | Cons                   |
| -------------- | ------------- | ------------------------------- | ----------------------------------- | ---------------------- |
| Jenkins        | Self-hosted   | Complex pipelines, full control | Highly customizable, plugins        | Requires maintenance   |
| GitHub Actions | Cloud         | GitHub repositories             | Native integration, free for public | GitHub-specific        |
| GitLab CI      | Integrated    | GitLab repositories             | Built-in, simple YAML               | GitLab-specific        |
| CircleCI       | Cloud         | Fast builds                     | Great Docker support                | Pricing can scale      |
| Azure DevOps   | Cloud/On-prem | Microsoft ecosystem             | Integrated with Azure               | Less common outside MS |

---

## 9️⃣ Interview Follow-Up Questions

### Q1: "What's the difference between CI and CD?"

**Answer**:
**CI (Continuous Integration)** focuses on integrating code frequently. Every commit triggers automated build and tests. The goal is catching integration issues early, before they compound.

**CD has two meanings**:

- **Continuous Delivery**: Code is always deployable. Automated deployment to staging, manual approval for production. You can deploy anytime but choose when.
- **Continuous Deployment**: Every change that passes tests automatically goes to production. No manual approval.

Most teams start with CI, then Continuous Delivery, then consider Continuous Deployment for low-risk changes.

### Q2: "How would you design a CI/CD pipeline for a microservices architecture?"

**Answer**:
Key considerations:

1. **Per-service pipelines**: Each microservice has its own pipeline. Changes to one service don't trigger builds for others.

2. **Shared libraries**: When shared library changes, trigger dependent service pipelines.

3. **Artifact versioning**: Use semantic versioning or commit SHA. Same artifact deployed to all environments.

4. **Deployment strategy**: Blue-green or canary for critical services, rolling for others.

5. **Integration testing**: After deployment, run contract tests and integration tests between services.

6. **Feature flags**: Use flags for gradual rollouts and instant rollbacks.

7. **Monitoring**: Post-deployment smoke tests and monitoring to catch issues quickly.

Example: Service A changes → Build → Test → Deploy to staging → Integration tests with Services B and C → Deploy to production → Smoke tests.

### Q3: "How do you handle database migrations in CI/CD?"

**Answer**:
Database migrations are tricky because they're stateful and often irreversible.

Approach:

1. **Separate migration pipeline**: Run migrations before application deployment.

2. **Backward compatibility**: Migrations must be backward compatible. Old code must work with new schema.

3. **Two-phase deployments**:

   - Phase 1: Deploy migration, deploy new code that supports both old and new schema
   - Phase 2: After verification, deploy code that only uses new schema

4. **Rollback strategy**: Keep rollback migrations. Test them in staging.

5. **Blue-green for databases**: Use read replicas. Migrate replica, verify, switch traffic.

6. **Feature flags**: Use flags to control which code path uses new schema.

Example: Add new column as nullable → Deploy code that writes to both old and new → Migrate data → Deploy code that only uses new → Remove old column.

### Q4: "What metrics do you track for CI/CD pipelines?"

**Answer**:
Key metrics:

1. **Build metrics**:

   - Build success rate (target: >95%)
   - Build duration (track trends, optimize slow builds)
   - Time to first feedback (commit to test results)

2. **Deployment metrics**:

   - Deployment frequency (how often you deploy)
   - Lead time (commit to production)
   - Mean time to recovery (MTTR) when deployments fail
   - Change failure rate (percentage of deployments causing incidents)

3. **Quality metrics**:

   - Test coverage
   - Test execution time
   - Flaky test rate

4. **Resource metrics**:
   - Pipeline cost per deployment
   - Agent utilization

These metrics help identify bottlenecks. For example, if lead time is high, investigate slow tests or manual approval gates.

### Q5: "How would you implement a rollback strategy?"

**Answer**:
Multiple layers of rollback:

1. **Application rollback**: Revert to previous version. In Kubernetes: `kubectl rollout undo deployment/myapp`. Keep previous Docker images tagged.

2. **Database rollback**: Run rollback migrations. This is risky, so prefer forward-only migrations when possible.

3. **Feature flag rollback**: Disable feature flag instantly. Code stays deployed but feature is off.

4. **Traffic rollback**: In blue-green, switch traffic back to previous environment.

5. **Infrastructure rollback**: If infrastructure change caused issue, rollback via Infrastructure as Code.

Best practice: Test rollback procedures in staging regularly. Automate where possible. Have runbooks for manual rollbacks.

For critical systems, I'd implement automated rollback triggers: if error rate spikes within 5 minutes of deployment, automatically rollback.

---

## 🔟 One Clean Mental Summary

CI/CD automates the software delivery pipeline. CI (Continuous Integration) runs automated builds and tests on every commit, catching integration issues early. CD (Continuous Delivery/Deployment) automates deployment, making code always deployable or automatically deploying to production.

The pipeline flow: commit → build → test → package → deploy. Each stage acts as a quality gate. If any stage fails, the pipeline stops and notifies the developer. Deployment strategies (blue-green, canary, rolling) minimize risk during releases.

The key insight: automation eliminates human error, provides fast feedback, and enables frequent, safe deployments. Feature flags add another safety layer, allowing instant rollbacks without code changes.
