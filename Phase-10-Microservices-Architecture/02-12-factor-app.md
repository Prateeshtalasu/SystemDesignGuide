# 12-Factor App

## 0️⃣ Prerequisites

Before diving into this topic, you need to understand:

- **Microservices Basics**: Understanding of distributed applications and service independence (Phase 10, Topic 1)
- **Environment Variables**: How applications read configuration from the environment
- **Cloud Computing**: Basic understanding of cloud platforms (AWS, Azure, GCP)
- **CI/CD Basics**: Continuous integration and deployment concepts
- **Logging Fundamentals**: How applications write and manage logs

**Quick refresher**: Microservices are independently deployable services. The 12-Factor App methodology is a set of principles that ensure your applications work well in cloud environments and can scale properly.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

In the early 2010s, developers were building applications for their laptops and then trying to deploy them to production servers. This led to a common problem: "It works on my machine."

Consider this scenario:

```java
// Developer's laptop
public class DatabaseConfig {
    private String dbUrl = "jdbc:mysql://localhost:3306/mydb";
    private String username = "root";
    private String password = "password123";
    // Hardcoded values that work on local machine
}
```

When this code reaches production:

- Production database is on a different host
- Credentials are different (and must be secure)
- Database might be MySQL, PostgreSQL, or something else
- Port might be different
- SSL might be required

The application breaks because it's tightly coupled to the development environment.

### What Systems Looked Like Before 12-Factor

Before the 12-Factor methodology, applications had these problems:

**Problem 1: Configuration Scattered Everywhere**

```java
// Configuration in code
String apiKey = "abc123xyz";

// Configuration in properties files
// application.properties
database.url=jdbc:mysql://localhost:3306/db

// Configuration in environment
$ export DATABASE_URL=...

// Configuration in config files
// config.xml
<database>
    <url>jdbc:mysql://prod-server:3306/prod_db</url>
</database>

// Which one wins? What gets overridden? Who knows?
```

**Problem 2: State Stored in the Filesystem**

```java
// Application writes files to local disk
File uploadDir = new File("/var/www/uploads");
File sessionStore = new File("/tmp/sessions");
File logFile = new File("/var/log/app.log");
```

When you scale horizontally (multiple instances), each instance has its own filesystem:

- User uploads file to Instance A
- User requests file from Instance B → File not found
- Sessions stored on Instance A are lost when Instance B handles the request

**Problem 3: Deployment Artifacts Not Portable**

```bash
# Developer builds on Windows
mvn clean package
# Creates: myapp.jar (Windows paths, Windows dependencies)

# Tries to run on Linux production server
java -jar myapp.jar
# Fails: Path separators wrong, dependencies missing
```

### What Breaks, Fails, or Becomes Painful

**At Scale (Multiple Instances):**

```
┌─────────────────┐
│  Instance 1     │  ← Has session data for User A
│  /tmp/sessions  │
└─────────────────┘
        ↓
   Load Balancer
        ↓
┌─────────────────┐
│  Instance 2     │  ← User A's request routed here
│  /tmp/sessions  │  ← No session data, user logged out
└─────────────────┘
```

**In Cloud Environments:**

- Your app assumes local filesystem persistence, but cloud containers are ephemeral
- You hardcode URLs, but cloud services use service discovery
- You assume single instance, but cloud scales automatically

**Real Examples of the Problem:**

**Heroku (2011)**: Developers were building apps that worked locally but failed in production. They noticed common patterns:

- Apps with hardcoded database URLs
- Apps writing to local filesystem
- Apps assuming specific ports
- Apps mixing code and configuration

They documented these problems and created the 12-Factor App methodology.

**Netflix**: Before adopting 12-Factor principles, their services were tightly coupled to specific infrastructure. Moving to AWS required significant rewrites. After adopting 12-Factor, they could deploy services to any cloud provider.

---

## 2️⃣ Intuition and Mental Model

### The Shipping Container Analogy

Think of the **12-Factor App methodology** like shipping container standards.

Before standardized shipping containers (1950s), cargo was loaded directly onto ships:

- Each cargo type needed custom handling
- Loading took days or weeks
- Ships had to be specially designed for specific cargo
- Moving cargo from ship to truck required manual unloading
- No standardization between ports

After standardized shipping containers:

- Any cargo goes into the same container
- Containers work on ships, trucks, trains (portability)
- Loading/unloading is fast and automated
- Containers are stateless (you don't care what's inside)
- Any port can handle any container

Similarly, before 12-Factor:

- Apps were custom-built for specific servers
- Deployment was manual and error-prone
- Apps couldn't move between environments easily
- Scaling required special configuration

After 12-Factor:

- Apps work the same way everywhere
- Deployment is automated and predictable
- Apps can run on any cloud platform
- Scaling is just launching more instances

### The Key Mental Model

**12-Factor Apps are like standardized containers:**

- **Portable**: Run anywhere (dev, staging, prod, any cloud)
- **Scalable**: Just add more instances, no special config
- **Stateless**: No local filesystem dependencies
- **Configurable**: Behavior changes via environment, not code changes
- **Disposable**: Can start/stop/restart gracefully

This analogy helps us understand why each factor exists: to make applications work like standardized containers that can run anywhere, scale horizontally, and be managed uniformly.

---

## 3️⃣ How It Works Internally

The 12-Factor App methodology consists of 12 principles. Let's understand how each one works:

### Factor I: Codebase

**One codebase tracked in revision control, many deploys**

```
┌──────────────────────────────────────┐
│      Git Repository                  │
│  (Single Source of Truth)            │
│                                      │
│  myapp/                              │
│  ├── src/                            │
│  ├── pom.xml                         │
│  └── README.md                       │
└──────────────────────────────────────┘
         │
         ├──→ Deploy to Development
         ├──→ Deploy to Staging
         └──→ Deploy to Production
```

**How it works:**

- One Git repository (or similar VCS)
- Multiple environments share the same codebase
- Different branches/tags for different versions
- Each deploy is a specific commit

**What breaks without it:**

- Code divergence between environments
- Can't reproduce production issues in dev
- Deployment becomes manual copy-paste

### Factor II: Dependencies

**Explicitly declare and isolate dependencies**

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>3.1.0</version>  <!-- Explicit version -->
    </dependency>
</dependencies>
```

**How it works:**

- Dependencies declared in build file (pom.xml, build.gradle, requirements.txt)
- Build system resolves and downloads dependencies
- Dependencies isolated per application (no shared system libraries)
- Versions pinned for reproducibility

### Factor III: Config

**Store config in the environment**

```java
// ❌ BAD: Config in code
String dbUrl = "jdbc:mysql://localhost:3306/mydb";

// ✅ GOOD: Config from environment
String dbUrl = System.getenv("DATABASE_URL");
```

**How it works:**

- Configuration (URLs, keys, credentials) read from environment variables
- Code stays the same, behavior changes via environment
- Different environments have different env vars
- Secrets never committed to codebase

### Factor IV: Backing Services

**Treat backing services as attached resources**

```
Application ──→ Database (attached resource)
Application ──→ Redis (attached resource)
Application ──→ S3 (attached resource)
Application ──→ SMTP Server (attached resource)
```

**How it works:**

- Databases, caches, queues are "attached resources"
- Accessed via URL/connection string from config
- Can swap local MySQL for managed RDS without code changes
- Local and production use same code, different backing services

### Factor V: Build, Release, Run

**Strictly separate build and run stages**

```
Build Stage:
  Source Code + Dependencies → Artifact (JAR file)

Release Stage:
  Artifact + Config → Release (immutable)

Run Stage:
  Release → Running Process
```

**How it works:**

- **Build**: Compile code, package dependencies → JAR file
- **Release**: Combine JAR + config → immutable release
- **Run**: Execute the release → running process
- Build happens once, release created per environment, run happens many times

### Factor VI: Processes

**Execute the app as one or more stateless processes**

```
Process 1: Web Server (handles HTTP requests)
Process 2: Worker (processes background jobs)
Process 3: Scheduler (runs cron jobs)
```

**How it works:**

- Application runs as one or more process types
- Each process is stateless (no local filesystem state)
- Processes share nothing (no shared memory)
- Any process can handle any request

### Factor VII: Port Binding

**Export services via port binding**

```java
// Application binds to port
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
        // Binds to port 8080 (or PORT env var)
    }
}
```

**How it works:**

- Application binds to a port (from PORT environment variable)
- Application IS the web server (embedded Tomcat/Jetty)
- No external web server (Apache, Nginx) required
- Platform routes traffic to the port

### Factor VIII: Concurrency

**Scale out via the process model**

```
Single Process:
┌─────────────────┐
│  App Instance   │
└─────────────────┘

Scale Out:
┌─────────────────┐
│  App Instance 1 │
└─────────────────┘
┌─────────────────┐
│  App Instance 2 │
└─────────────────┘
┌─────────────────┐
│  App Instance 3 │
└─────────────────┘
```

**How it works:**

- Scale horizontally (more processes) not vertically (bigger processes)
- Process manager (systemd, Kubernetes) manages processes
- Load balancer distributes traffic across processes
- Each process is independent

### Factor IX: Disposability

**Maximize robustness with fast startup and graceful shutdown**

```java
// Graceful shutdown
@PreDestroy
public void cleanup() {
    // Stop accepting new requests
    // Finish processing current requests
    // Close connections
    // Release resources
}
```

**How it works:**

- Processes start quickly (< 1 second ideally)
- Processes shut down gracefully (SIGTERM handling)
- Processes can be killed and restarted without data loss
- No reliance on filesystem state

### Factor X: Dev/Prod Parity

**Keep development, staging, and production as similar as possible**

```
Development:  Docker container, PostgreSQL, Redis
Staging:      Docker container, PostgreSQL, Redis
Production:   Docker container, PostgreSQL, Redis

Same technology stack, different scale
```

**How it works:**

- Use same backing services (PostgreSQL in dev and prod, not SQLite in dev)
- Use same OS and runtime versions
- Minimize differences (time gaps, personnel gaps, tool gaps)
- Production-like local environment

### Factor XI: Logs

**Treat logs as event streams**

```java
// Application writes to stdout
logger.info("User logged in: {}", userId);
// Output: 2024-01-15 10:30:45 INFO User logged in: 12345

// Platform captures stdout/stderr
// Routes to log aggregation service (CloudWatch, ELK, etc.)
```

**How it works:**

- Application writes logs to stdout/stderr (not files)
- Platform (container orchestrator) captures stdout/stderr
- Platform routes logs to aggregation system
- Application doesn't manage log files, rotation, or storage

### Factor XII: Admin Processes

**Run admin/management tasks as one-off processes**

```bash
# Admin task as one-off process
java -jar app.jar migrate-database
java -jar app.jar seed-data
java -jar app.jar generate-report

# Not part of long-running web process
```

**How it works:**

- Admin tasks (migrations, data fixes, reports) run as separate processes
- Use same codebase and config as main app
- Run once and exit (not long-running)
- Can run in same environment as app

---

## 4️⃣ Simulation-First Explanation

Let's build a simple 12-Factor application from scratch to see how it works:

### Starting Point: Non-12-Factor App

```java
// App.java - BAD EXAMPLE
public class App {
    public static void main(String[] args) {
        // Hardcoded config
        String dbUrl = "jdbc:mysql://localhost:3306/mydb";
        String dbUser = "root";
        String dbPass = "password";

        // Writes to local filesystem
        File logFile = new File("/var/log/app.log");
        PrintWriter log = new PrintWriter(logFile);

        // Hardcoded port
        Server server = new Server(8080);

        // State in memory
        Map<String, String> sessions = new HashMap<>();

        server.start();
    }
}
```

**Problems:**

1. Config hardcoded (can't change for different environments)
2. Writes to filesystem (breaks with multiple instances)
3. Hardcoded port (can't change per environment)
4. In-memory state (lost on restart, can't share across instances)

### Converting to 12-Factor: Step by Step

**Step 1: Factor III - Config from Environment**

```java
// App.java - Config from environment
public class App {
    public static void main(String[] args) {
        // Read from environment
        String dbUrl = System.getenv("DATABASE_URL");
        String dbUser = System.getenv("DATABASE_USER");
        String dbPass = System.getenv("DATABASE_PASSWORD");

        if (dbUrl == null) {
            throw new IllegalArgumentException("DATABASE_URL must be set");
        }

        // Use config...
    }
}
```

**Run with environment:**

```bash
export DATABASE_URL=jdbc:mysql://localhost:3306/mydb
export DATABASE_USER=root
export DATABASE_PASSWORD=password
java -jar app.jar
```

**Step 2: Factor XI - Logs to stdout**

```java
// App.java - Logs to stdout
import java.util.logging.Logger;

public class App {
    private static final Logger logger = Logger.getLogger(App.class.getName());

    public static void main(String[] args) {
        // Log to stdout (not file)
        logger.info("Application starting");
        logger.info("Database URL: " + System.getenv("DATABASE_URL"));

        // No file writing
    }
}
```

**Step 3: Factor VII - Port Binding**

```java
// App.java - Port from environment
public class App {
    public static void main(String[] args) {
        // Port from environment (default 8080)
        int port = Integer.parseInt(
            System.getenv().getOrDefault("PORT", "8080")
        );

        Server server = new Server(port);
        server.start();
    }
}
```

**Step 4: Factor VI - Stateless Processes**

```java
// App.java - Stateless (no in-memory sessions)
public class App {
    // ❌ Remove: Map<String, String> sessions = new HashMap<>();

    // ✅ Use external session store (Redis from config)
    private SessionStore sessionStore;

    public App() {
        String redisUrl = System.getenv("REDIS_URL");
        this.sessionStore = new RedisSessionStore(redisUrl);
    }
}
```

### Complete 12-Factor Application

```java
// Application.java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// application.yml (defaults only, no secrets)
server:
  port: ${PORT:8080}  # From environment, default 8080

spring:
  datasource:
    url: ${DATABASE_URL}  # Must be set in environment
    username: ${DATABASE_USER}
    password: ${DATABASE_PASSWORD}

logging:
  level:
    root: ${LOG_LEVEL:INFO}  # From environment
```

**Deploy to Development:**

```bash
export PORT=8080
export DATABASE_URL=jdbc:mysql://dev-db:3306/mydb
export DATABASE_USER=dev_user
export DATABASE_PASSWORD=dev_pass
export LOG_LEVEL=DEBUG
java -jar app.jar
```

**Deploy to Production:**

```bash
export PORT=8080
export DATABASE_URL=jdbc:mysql://prod-db:3306/mydb
export DATABASE_USER=prod_user
export DATABASE_PASSWORD=${SECRET_PASSWORD}  # From secrets manager
export LOG_LEVEL=INFO
java -jar app.jar
```

**Same code, different behavior via environment variables.**

---

## 5️⃣ How Engineers Actually Use This in Production

### Real-World Implementation: Spring Boot Application

**Heroku Deployment:**

```java
// Application automatically reads PORT from environment
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
        // Spring Boot reads PORT env var automatically
    }
}
```

```bash
# Heroku sets PORT automatically
$ heroku create myapp
$ git push heroku main
# Heroku builds, releases, and runs
# Sets PORT=random, DATABASE_URL=managed_postgres_url
```

**Kubernetes Deployment:**

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3 # Factor VIII: Concurrency (scale processes)
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          env:
            - name: PORT
              value: "8080"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          # Factor XI: Logs to stdout, Kubernetes captures
```

**Netflix Implementation:**

Netflix uses 12-Factor principles for all their microservices:

1. **Config Management**: All config in environment variables, managed by their internal config service
2. **Backing Services**: Services connect to databases, caches, queues via URLs from service discovery
3. **Logs**: All services write to stdout, captured by their logging infrastructure (ELK stack)
4. **Disposability**: Services designed to start in < 1 second, handle graceful shutdown
5. **Concurrency**: Auto-scaling based on load (more processes, not bigger processes)

**Uber Implementation:**

Uber's microservices follow 12-Factor:

1. **Codebase**: Single Git repo per service, multiple deploys (dev, staging, prod, regions)
2. **Dependencies**: Maven/Gradle with locked versions, Docker containers ensure consistency
3. **Config**: Config service (internal) provides environment-specific configuration
4. **Processes**: Kubernetes manages process lifecycle, services are stateless
5. **Port Binding**: Services bind to ports, Kubernetes service mesh routes traffic

**Production War Stories:**

**Story 1: The Hardcoded URL**

A startup hardcoded their database URL in code. When they moved from AWS us-east-1 to us-west-2, the application broke. They had to redeploy with new code. After adopting 12-Factor (config in environment), they changed DATABASE_URL and restarted, no code change needed.

**Story 2: The Filesystem Session Store**

An application stored sessions in `/tmp/sessions`. It worked fine with one instance. When they scaled to two instances behind a load balancer, users were constantly logged out because sessions were stored on Instance A but requests went to Instance B. They moved to Redis (backing service), problem solved.

**Story 3: The Long Startup Time**

A Java application took 2 minutes to start because it loaded all data into memory on startup. When Kubernetes restarted pods (rolling updates), there was a 2-minute window with reduced capacity. They optimized startup (lazy loading, async initialization) to start in < 5 seconds, following Factor IX (Disposability).

---

## 6️⃣ How to Implement or Apply It

### Step-by-Step: Converting an Existing App to 12-Factor

**Step 1: Extract Configuration**

```java
// Before
public class DatabaseConfig {
    private String url = "jdbc:mysql://localhost:3306/mydb";
    private String user = "root";
    private String password = "password";
}

// After
@Configuration
public class DatabaseConfig {
    @Value("${DATABASE_URL}")
    private String url;

    @Value("${DATABASE_USER}")
    private String user;

    @Value("${DATABASE_PASSWORD}")
    private String password;
}
```

**Step 2: Move Logs to stdout**

```java
// Before
File logFile = new File("/var/log/app.log");
Logger logger = new FileLogger(logFile);

// After
Logger logger = LoggerFactory.getLogger(MyClass.class);
// Logback configured to write to stdout
```

**logback.xml:**

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    <root level="INFO">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

**Step 3: Make Stateless**

```java
// Before: In-memory state
private Map<String, Session> sessions = new HashMap<>();

// After: External backing service
@Service
public class SessionService {
    @Autowired
    private RedisTemplate<String, Session> redis;

    public Session getSession(String sessionId) {
        return redis.opsForValue().get("session:" + sessionId);
    }
}
```

**Step 4: Use Environment Variables**

**application.yml (no secrets):**

```yaml
server:
  port: ${PORT:8080}

spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USER}
    password: ${DATABASE_PASSWORD}
  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}

logging:
  level:
    root: ${LOG_LEVEL:INFO}
```

**Step 5: Dockerfile (12-Factor)**

```dockerfile
# Build stage
FROM maven:3.8-openjdk-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Run stage
FROM openjdk:17-jre-slim
WORKDIR /app
COPY --from=build /app/target/myapp.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
# PORT, DATABASE_URL, etc. come from environment
```

**Step 6: Docker Compose (Dev Environment)**

```yaml
version: "3.8"
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - PORT=8080
      - DATABASE_URL=jdbc:mysql://db:3306/mydb
      - DATABASE_USER=root
      - DATABASE_PASSWORD=password
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - LOG_LEVEL=DEBUG
    depends_on:
      - db
      - redis

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: mydb
      MYSQL_ROOT_PASSWORD: password

  redis:
    image: redis:7-alpine
```

**Step 7: Kubernetes Deployment**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  PORT: "8080"
  LOG_LEVEL: "INFO"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_URL: "jdbc:mysql://db-service:3306/mydb"
  DATABASE_USER: "app_user"
  DATABASE_PASSWORD: "secret_password"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
          ports:
            - containerPort: 8080
```

**Verification:**

```bash
# Check environment variables
kubectl exec deployment/myapp -- env | grep DATABASE

# Check logs (stdout)
kubectl logs deployment/myapp

# Scale processes
kubectl scale deployment/myapp --replicas=5
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Tradeoffs

**Advantages:**

- **Portability**: Run anywhere (local, cloud, any region)
- **Scalability**: Horizontal scaling is straightforward
- **Maintainability**: Clear separation of code and config
- **Security**: Secrets not in codebase
- **Reliability**: Stateless processes can restart without data loss

**Disadvantages:**

- **Complexity**: More moving parts (backing services, config management)
- **Latency**: External backing services add network calls
- **Cost**: More infrastructure (databases, caches, queues) instead of local resources
- **Learning Curve**: Team needs to understand 12-Factor principles

### Common Pitfalls

**Pitfall 1: Partial Adoption**

```java
// Mixed approach (BAD)
String dbUrl = System.getenv("DATABASE_URL");  // ✅ Good
File logFile = new File("/var/log/app.log");   // ❌ Bad
```

**Solution**: Adopt all 12 factors consistently. Partial adoption creates confusion and doesn't provide full benefits.

**Pitfall 2: Config in Multiple Places**

```java
// Config from environment
String dbUrl = System.getenv("DATABASE_URL");

// Also in properties file
@Value("${database.url}") String dbUrl2;

// Which one wins? Confusion!
```

**Solution**: Use environment variables as source of truth. Properties files only for defaults.

**Pitfall 3: Assuming Local Filesystem**

```java
// BAD: Assumes local filesystem
File uploadDir = new File("/uploads");
userFile.save(uploadDir);

// In cloud/containers, /uploads doesn't persist
```

**Solution**: Use backing service (S3, Azure Blob) for file storage.

**Pitfall 4: In-Memory State**

```java
// BAD: State in memory
private Map<String, Cart> shoppingCarts = new ConcurrentHashMap<>();

// Lost on restart, can't share across instances
```

**Solution**: Use external store (Redis, database) for state.

**Pitfall 5: Long Startup Time**

```java
// BAD: Loads everything on startup
@PostConstruct
public void init() {
    loadAllProducts();  // Takes 30 seconds
    precomputeReports();  // Takes 20 seconds
}
// Startup: 50 seconds
```

**Solution**: Lazy load, async initialization, follow Factor IX (Disposability).

### Performance Considerations

**Config Lookup Overhead:**

```java
// BAD: Reading env var on every request
public String getDatabaseUrl() {
    return System.getenv("DATABASE_URL");  // System call each time
}

// GOOD: Cache in application context
@Configuration
public class Config {
    @Value("${DATABASE_URL}")
    private String databaseUrl;  // Read once at startup
}
```

**Backing Service Latency:**

Stateless processes require external backing services, which add network latency. Use connection pooling, caching, and async operations where appropriate.

### Security Considerations

**Secrets Management:**

```java
// ❌ BAD: Secrets in code or config files
String apiKey = "sk_live_1234567890";

// ✅ GOOD: Secrets from environment (set by platform)
String apiKey = System.getenv("API_KEY");

// ✅ BETTER: Secrets from secrets manager (AWS Secrets Manager, Vault)
String apiKey = secretsManager.getSecret("api-key");
```

**Environment Variable Exposure:**

Be careful not to log environment variables:

```java
// ❌ BAD: Logs secrets
logger.info("Database URL: " + System.getenv("DATABASE_URL"));

// ✅ GOOD: Log structure, not values
logger.info("Database configured");
```

---

## 8️⃣ When NOT to Use This

### Anti-Patterns and Misuse Cases

**Don't use 12-Factor for:**

**1. Desktop Applications**

12-Factor is designed for web applications and services. Desktop apps have different requirements (local filesystem is appropriate, single user, no horizontal scaling).

**2. Embedded Systems**

IoT devices, embedded systems have resource constraints and different deployment models. 12-Factor assumes cloud/container environments.

**3. Batch Jobs (Sometimes)**

Long-running batch jobs that process large files might legitimately use local filesystem. However, consider if they can be adapted (stream processing, cloud storage).

### Situations Where This is Overkill

**Small Internal Tools:**

If you're building a small tool for internal use, running on a single server, strict 12-Factor might be overkill. However, the principles still apply (config from environment, logs to stdout).

**Proof of Concept:**

For rapid prototyping, you might hardcode values. But if the PoC becomes production, refactor to 12-Factor early.

### Better Alternatives for Specific Scenarios

**Serverless Functions:**

Serverless (AWS Lambda, Azure Functions) has different constraints. Some 12-Factor principles apply (stateless, config from environment), but port binding doesn't (functions don't bind ports).

**Edge Computing:**

Edge computing (CDN edge functions, CloudFlare Workers) has different deployment models. Adapt 12-Factor principles rather than applying blindly.

### Signs You've Chosen Wrong Approach

- You're fighting the platform (trying to use local filesystem in serverless)
- Config changes require code deployments
- Scaling requires manual intervention
- Different behavior in dev vs prod for non-scale reasons
- Can't run multiple instances
- Processes take minutes to start

---

## 9️⃣ Comparison with Alternatives

### 12-Factor vs Traditional Deployment

**Traditional Deployment:**

```
Development:
  - Code + Config in same files
  - Assumes local filesystem
  - Single instance
  - Manual deployment

Production:
  - Different code/config structure
  - Shared filesystem (NFS)
  - Multiple instances with sticky sessions
  - Manual deployment
```

**12-Factor:**

```
Development:
  - Code separate from config (env vars)
  - Backing services (containers)
  - Single instance (same as prod structure)
  - Automated deployment

Production:
  - Same structure as dev
  - Backing services (managed services)
  - Multiple instances (stateless)
  - Automated deployment
```

**When to choose 12-Factor**: Cloud-native applications, microservices, scalable systems

**When to choose Traditional**: Legacy systems, on-premise deployments, single-instance applications

### 12-Factor vs Configuration Management Tools

**Configuration Management (Ansible, Chef, Puppet):**

- Manages server state (installs packages, configures services)
- Works at infrastructure level
- Good for traditional deployments

**12-Factor:**

- Application-level methodology
- Works with any infrastructure
- Good for cloud-native applications

**They complement each other**: Use configuration management for infrastructure, 12-Factor for applications.

### Migration Path from Traditional to 12-Factor

**Step 1: Extract Configuration**

Move hardcoded values to environment variables.

**Step 2: Externalize State**

Move sessions, caches, file storage to backing services.

**Step 3: Containerize**

Package application in container (Docker) for consistency.

**Step 4: Update Deployment**

Use CI/CD to build, release, run. Deploy to container platform (Kubernetes, ECS).

**Step 5: Scale Horizontally**

Remove sticky sessions, ensure statelessness, scale processes.

---

## 🔟 Interview Follow-up Questions WITH Answers

### L4 Level Questions

**Q1: What is the 12-Factor App methodology?**

**Answer**: The 12-Factor App is a methodology for building software-as-a-service applications. It consists of 12 principles that ensure applications are portable, scalable, and suitable for cloud deployment. Key principles include storing config in environment variables, treating backing services as attached resources, executing apps as stateless processes, and treating logs as event streams.

**Q2: Why store configuration in environment variables instead of config files?**

**Answer**: Environment variables allow the same codebase to run in different environments (dev, staging, prod) with different configurations without code changes. Config files often get committed to version control with secrets, or require different files per environment leading to divergence. Environment variables are set by the deployment platform, keeping secrets out of code and making deployments more secure and flexible.

**Q3: What does "stateless processes" mean in 12-Factor?**

**Answer**: Stateless processes don't store any data locally (no filesystem state, no in-memory state that persists). Any request can be handled by any process instance. State is stored in backing services (databases, caches). This enables horizontal scaling (adding more processes) because any process can handle any request, and processes can be restarted or replaced without data loss.

### L5 Level Questions

**Q4: How do you handle file uploads in a 12-Factor app?**

**Answer**: In a 12-Factor app, you don't store files on the local filesystem because it's ephemeral and not shared across instances. Instead, upload files directly to a backing service like S3, Azure Blob Storage, or Google Cloud Storage. The application generates a URL to the uploaded file and stores that URL in the database. This allows any instance to serve the file, and files persist even if application instances are replaced.

**Q5: Explain the build, release, run separation in 12-Factor.**

**Answer**:

- **Build**: Compiles code and packages dependencies into an artifact (e.g., JAR file). This happens once per code change.
- **Release**: Takes the build artifact and combines it with configuration for a specific environment, creating an immutable release. Each environment gets its own release.
- **Run**: Executes the release as a running process. The same release can run multiple times (multiple process instances).

This separation ensures builds are reproducible, releases are immutable (can't change config in running process), and the same release runs identically in dev and prod (just different config).

**Q6: How does 12-Factor handle logging?**

**Answer**: 12-Factor apps write logs to stdout/stderr as unbuffered event streams. The application doesn't manage log files, rotation, or storage. The execution environment (container platform, process manager) captures stdout/stderr and routes it to a log aggregation system (CloudWatch, ELK stack, Splunk). This keeps the app simple and allows the platform to handle log management, aggregation, and analysis.

### L6 Level Questions

**Q7: You're migrating a legacy monolith to 12-Factor. The app stores user sessions in-memory and uses local filesystem for temporary files. How do you approach this?**

**Answer**:

1. **Sessions**: Move to external backing service. Options:

   - Redis (fast, in-memory, shared across instances)
   - Database (persistent, slower)
   - JWT tokens (stateless, no server-side storage)

2. **Temporary Files**:

   - For truly temporary files: Use in-memory processing (streams, byte arrays) instead of filesystem
   - For files that need persistence: Upload to object storage (S3) immediately, process from there
   - For file processing: Use message queues (user uploads → queue → worker processes → store result in database/S3)

3. **Migration Strategy**:
   - Extract session management to a service/abstraction
   - Implement Redis adapter alongside in-memory adapter
   - Feature flag to switch between implementations
   - Deploy with in-memory (existing behavior)
   - Switch to Redis in canary deployment
   - Monitor for issues, rollback if needed
   - Remove in-memory implementation once stable

**Q8: How do you handle database migrations in a 12-Factor app with multiple instances?**

**Answer**: Database migrations should run as one-off admin processes (Factor XII), not as part of the application startup. Process:

1. Deploy new code (with migration scripts) but don't start new instances yet
2. Run migration as separate process: `java -jar app.jar migrate`
3. Verify migration success
4. Start new application instances (they expect new schema)
5. If migration fails, don't start new instances, rollback if needed

For zero-downtime:

- Write migrations that are backward-compatible (add columns as nullable, don't drop columns immediately)
- Deploy code that works with both old and new schema
- Run migration
- Deploy code that requires new schema
- Later, remove old schema support

Use tools like Flyway or Liquibase that track which migrations have run, ensuring idempotency.

**Q9: A 12-Factor app needs to process large files (GBs). How do you handle this given the stateless process constraint?**

**Answer**: Large file processing in stateless processes:

1. **Stream Processing**: Don't load entire file into memory. Process in chunks/streams.

   ```java
   InputStream input = s3Client.getObject(bucket, key).getObjectContent();
   // Process stream, not entire file
   ```

2. **External Processing**:

   - Upload file to object storage (S3)
   - Send message to queue with file reference
   - Worker processes (separate process type) pick up messages
   - Workers stream process files from S3
   - Store results in database/object storage

3. **Chunked Processing**: Break large files into chunks, process chunks independently (map-reduce pattern).

4. **Managed Services**: Use managed services (AWS Glue, Azure Data Factory) for large-scale data processing instead of application processes.

The key is: application processes remain stateless and lightweight. Heavy processing happens in specialized worker processes or managed services, not in web request handlers.

---

## 1️⃣1️⃣ One Clean Mental Summary

The 12-Factor App methodology is a set of principles that make applications work like standardized shipping containers: portable (run anywhere), scalable (just add more instances), and manageable (consistent behavior). The core idea is separation: code is separate from configuration (stored in environment), applications are separate from state (stored in backing services), and processes are stateless (any process can handle any request). This enables applications to run consistently across development, staging, and production, scale horizontally by adding more processes, and be deployed and managed uniformly in cloud environments. Think of it as making your application cloud-native by design, not by accident.
