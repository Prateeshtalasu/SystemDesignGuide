# Service Discovery in Microservices

## 0️⃣ Prerequisites

Before diving into this topic, you need to understand:

- **Service Discovery Basics**: Client-side vs server-side discovery, service registries (Phase 2, Topic 13)
- **Microservices Architecture**: Understanding of service independence and communication (Phase 10, Topic 1)
- **Load Balancing**: How traffic is distributed across instances (Phase 2, Topic 6)
- **Health Checks**: How services report their health status
- **DNS Fundamentals**: Basic understanding of DNS resolution

**Quick refresher**: Service discovery allows services to find each other dynamically. In microservices, this is critical because instances are created and destroyed frequently, and IP addresses change constantly. This topic focuses on service discovery patterns specific to microservices architecture.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

In a microservices architecture, you have many services that need to communicate:

```
Order Service needs to call:
  - Payment Service
  - Inventory Service
  - Shipping Service

Payment Service needs to call:
  - Order Service (for order details)
  - Billing Service
  - Notification Service
```

Each service runs on multiple instances for scalability and reliability:

```
Payment Service Instances:
  - payment-service-1: 192.168.1.10:8080
  - payment-service-2: 192.168.1.11:8080
  - payment-service-3: 192.168.1.12:8080 (just started)
  - payment-service-4: 192.168.1.13:8080 (just died)
```

**The Problem**: How does Order Service find Payment Service instances when:
- Instances are added/removed dynamically (auto-scaling)
- IP addresses change (containers restart, pods reschedule)
- Instances fail and need to be avoided
- New versions are deployed (blue-green, canary)

### What Systems Looked Like Before Service Discovery in Microservices

**Hardcoded Configuration (Doesn't Scale):**

```java
// Order Service Configuration
@Configuration
public class ServiceConfig {
    @Value("${payment.service.url}")
    private String paymentServiceUrl;  // http://192.168.1.10:8080
    
    @Value("${inventory.service.url}")
    private String inventoryServiceUrl;  // http://192.168.1.11:8080
}
```

**Problems:**
1. Payment Service scales to 5 instances → Can't use them all
2. Payment Service instance dies → Requests fail until config updated
3. New Payment Service instance starts → Not used until config updated
4. Payment Service moves to new host → All services need config update
5. With 50 services, managing configs becomes impossible

### What Breaks Without Service Discovery

**At Microservices Scale:**

```
50 services × 3 instances each = 150 endpoints
Each service calls 5 other services on average
= 250 hardcoded URLs to manage
× 3 environments (dev, staging, prod)
= 750 configuration values
× Changes every deployment
= Operational nightmare
```

**Real-World Failures:**

- **Instance Failure**: One Payment Service instance dies. Order Service still sends 25% of traffic to dead instance. 25% of payment requests fail.

- **Auto-Scaling**: Payment Service scales from 3 to 10 instances during Black Friday. Order Service only knows about 3 instances, can't use the new 7 instances. Underutilization.

- **Deployment**: Payment Service deploys new version. Old instances shut down, new instances start with different IPs. Order Service has stale IPs. Downtime.

### Real Examples of the Problem

**Netflix (Pre-Eureka)**: 
- Hundreds of services with hardcoded IPs
- Deployments required coordination across teams
- Instance failures caused cascading failures
- Created Eureka to solve this (now open-source)

**Uber (Early Days)**:
- Services hardcoded service locations
- Scaling required manual config updates across services
- Multi-region deployments were complex
- Adopted Consul for service discovery

**Amazon (Pre-AWS)**: 
- Internal services used hardcoded endpoints
- Service changes required cross-team coordination
- Created internal service discovery (precursor to AWS Service Discovery)

---

## 2️⃣ Intuition and Mental Model

### The Restaurant Reservation System Analogy

Think of **service discovery** like a restaurant reservation system.

**Without Service Discovery (Direct Calls):**

```
You want to eat at "Pizza Place"
You call the phone number you memorized: 555-0100
- Phone busy → Try again later
- Restaurant moved → Number doesn't work
- New location opened → You don't know about it
```

**With Service Discovery (Reservation System):**

```
You want to eat at "Pizza Place"
You call the reservation system: "Where is Pizza Place?"
System tells you:
  - Location 1: Downtown, 3 tables available
  - Location 2: Uptown, 5 tables available (just opened)
  - Location 3: Closed (kitchen issue)

You pick Location 2 (most availability)
You make reservation directly with Location 2
```

**Service Discovery Components:**

- **Service Registry**: The reservation system (knows all locations)
- **Service Registration**: Restaurants register their locations and availability
- **Service Discovery**: You query the system to find locations
- **Health Checks**: System knows which locations are open/closed
- **Load Balancing**: System tells you which location has availability

### The Key Mental Model

**Service discovery is like a dynamic phone book that updates in real-time:**

- Services register themselves (like restaurants registering locations)
- Registry tracks all instances and their health (like tracking which locations are open)
- Clients query registry to find services (like calling reservation system)
- Registry provides multiple instances for load balancing (like multiple restaurant locations)
- Registry removes dead instances automatically (like removing closed locations)

This analogy helps us understand:
- Why registration is needed (restaurants must register)
- Why health checks matter (system tracks which are open)
- Why multiple instances help (multiple locations for capacity)
- Why it's dynamic (locations open/close, services start/stop)

---

## 3️⃣ How It Works Internally

Service discovery in microservices follows these patterns (see Phase 2, Topic 13 for detailed basics). Here we focus on microservices-specific aspects:

### Pattern 1: Client-Side Discovery (Eureka, Consul)

```
┌─────────────────────────────────────────────────────────┐
│              CLIENT-SIDE DISCOVERY                       │
└─────────────────────────────────────────────────────────┘

1. Service Registration:
   Payment Service Instance starts
   → Registers with Eureka: "I'm payment-service, at 192.168.1.10:8080"
   → Sends heartbeat every 30 seconds

2. Service Discovery:
   Order Service needs Payment Service
   → Queries Eureka: "Give me payment-service instances"
   → Eureka returns: [192.168.1.10:8080, 192.168.1.11:8080, 192.168.1.12:8080]
   → Order Service caches list
   → Order Service picks instance (round-robin, random, etc.)
   → Order Service calls instance directly

3. Health Monitoring:
   Payment Service instance dies
   → Stops sending heartbeat
   → After 90 seconds, Eureka removes from registry
   → Order Service refreshes cache
   → No longer routes to dead instance
```

**Key Characteristics:**
- Client queries registry and caches results
- Client does load balancing
- Client calls service directly (no extra hop)
- Requires client-side library (Eureka client, Consul client)

### Pattern 2: Server-Side Discovery (Kubernetes Services)

```
┌─────────────────────────────────────────────────────────┐
│              SERVER-SIDE DISCOVERY                       │
└─────────────────────────────────────────────────────────┘

1. Service Registration:
   Payment Service Pod starts
   → Kubernetes tracks pod IP automatically
   → Kubernetes Service (payment-service) selects pods by labels
   → Service has stable DNS name: payment-service.namespace.svc.cluster.local

2. Service Discovery:
   Order Service needs Payment Service
   → Calls: http://payment-service/api/payments
   → DNS resolves to Service IP (virtual IP)
   → Kubernetes kube-proxy routes to healthy pod
   → Order Service doesn't know about individual pods

3. Health Monitoring:
   Payment Service Pod dies
   → Kubernetes removes from Service endpoint list
   → kube-proxy stops routing to dead pod
   → Order Service sees no change (still calls payment-service DNS)
```

**Key Characteristics:**
- Client calls stable DNS name
- Platform (Kubernetes) handles discovery and load balancing
- Client is simpler (no library needed)
- Extra hop through service proxy

### Pattern 3: Service Mesh Discovery (Istio, Linkerd)

```
┌─────────────────────────────────────────────────────────┐
│              SERVICE MESH DISCOVERY                      │
└─────────────────────────────────────────────────────────┘

Order Service Pod
  └── Sidecar Proxy (Envoy)
      └── Service Registry (Pilot/Control Plane)
          └── Tracks all service instances

1. Service Registration:
   Payment Service Pod starts
   → Sidecar proxy registers with control plane
   → Control plane updates service registry

2. Service Discovery:
   Order Service calls Payment Service
   → Request intercepted by sidecar proxy
   → Sidecar queries control plane for Payment Service instances
   → Sidecar routes to healthy instance
   → Order Service code doesn't know about discovery

3. Health Monitoring:
   Control plane tracks health via sidecar health reports
   → Dead instances removed from routing
   → Sidecar proxies updated automatically
```

**Key Characteristics:**
- Discovery handled by sidecar proxy
- Application code unchanged
- Advanced features (circuit breaking, retries, metrics)
- Requires service mesh infrastructure

---

## 4️⃣ Simulation-First Explanation

Let's trace a request through service discovery in a microservices setup:

### Scenario: Order Service Calls Payment Service

**Setup:**
- Order Service: 2 instances
- Payment Service: 3 instances
- Eureka Service Registry

**Step 1: Service Registration**

```java
// Payment Service Instance 1 starts
@SpringBootApplication
@EnableEurekaClient
public class PaymentServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentServiceApplication.class, args);
        // Eureka client automatically registers:
        // Service: payment-service
        // Instance: payment-service-1
        // IP: 192.168.1.10
        // Port: 8080
        // Status: UP
    }
}
```

Eureka Registry state:
```
payment-service:
  - instance-1: 192.168.1.10:8080 (UP, last heartbeat: now)
  - instance-2: 192.168.1.11:8080 (UP, last heartbeat: now)
  - instance-3: 192.168.1.12:8080 (UP, last heartbeat: now)
```

**Step 2: Service Discovery**

```java
// Order Service needs to call Payment Service
@Service
public class OrderService {
    @Autowired
    private RestTemplate restTemplate;  // Configured with Eureka
    
    public PaymentResult processPayment(Order order) {
        // RestTemplate configured with Eureka client
        // Automatically resolves "payment-service" to actual instances
        PaymentResult result = restTemplate.postForObject(
            "http://payment-service/api/payments",  // Service name, not IP
            order,
            PaymentResult.class
        );
        return result;
    }
}
```

**What happens internally:**

1. RestTemplate calls Eureka client library
2. Eureka client queries Eureka: "Give me payment-service instances"
3. Eureka returns: `[192.168.1.10:8080, 192.168.1.11:8080, 192.168.1.12:8080]`
4. Eureka client caches list (refreshes every 30 seconds)
5. Eureka client picks instance using Ribbon (round-robin): `192.168.1.11:8080`
6. RestTemplate calls: `http://192.168.1.11:8080/api/payments`

**Step 3: Instance Failure**

```
Payment Service Instance 2 dies (192.168.1.11:8080)
  → Stops sending heartbeat to Eureka
  → After 90 seconds, Eureka marks as DOWN
  → Eureka removes from registry

Eureka Registry state:
payment-service:
  - instance-1: 192.168.1.10:8080 (UP)
  - instance-3: 192.168.1.12:8080 (UP)
  // instance-2 removed

Order Service makes another call:
  → Eureka client cache expires (30 seconds)
  → Queries Eureka again
  → Gets updated list: [192.168.1.10:8080, 192.168.1.12:8080]
  → Picks instance: 192.168.1.10:8080
  → Dead instance no longer used
```

**Step 4: New Instance Added**

```
Payment Service Instance 4 starts (192.168.1.13:8080)
  → Registers with Eureka
  → Eureka adds to registry

Eureka Registry state:
payment-service:
  - instance-1: 192.168.1.10:8080 (UP)
  - instance-3: 192.168.1.12:8080 (UP)
  - instance-4: 192.168.1.13:8080 (UP)  ← New

Order Service makes call:
  → After cache refresh
  → Gets all 3 instances
  → New instance available for load balancing
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Real-World Implementation: Spring Cloud Eureka

**Architecture:**

```
Eureka Server (Service Registry)
  ├── payment-service instances
  ├── order-service instances
  ├── inventory-service instances
  └── shipping-service instances
```

**Eureka Server:**

```java
// Eureka Server Application
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# application.yml
server:
  port: 8761

eureka:
  instance:
    hostname: eureka-server
  client:
    register-with-eureka: false  # Server doesn't register itself
    fetch-registry: false
```

**Service Client (Payment Service):**

```java
// Payment Service
@SpringBootApplication
@EnableEurekaClient
public class PaymentServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentServiceApplication.class, args);
    }
}
```

```yaml
# application.yml
spring:
  application:
    name: payment-service  # Service name in registry

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90
```

**Service Consumer (Order Service):**

```java
// Order Service - Using RestTemplate with Eureka
@Configuration
public class RestTemplateConfig {
    @Bean
    @LoadBalanced  // Enables Eureka service discovery
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
public class OrderService {
    @Autowired
    private RestTemplate restTemplate;
    
    public PaymentResult processPayment(Order order) {
        // "payment-service" is resolved by Eureka + Ribbon
        return restTemplate.postForObject(
            "http://payment-service/api/payments",
            order,
            PaymentResult.class
        );
    }
}
```

**Service Consumer (Using Feign Client):**

```java
// Feign Client (declarative REST client)
@FeignClient(name = "payment-service")
public interface PaymentServiceClient {
    @PostMapping("/api/payments")
    PaymentResult processPayment(@RequestBody Order order);
}

@Service
public class OrderService {
    @Autowired
    private PaymentServiceClient paymentClient;
    
    public PaymentResult processPayment(Order order) {
        return paymentClient.processPayment(order);
    }
}
```

### Kubernetes Service Discovery

**Service Definition:**

```yaml
# payment-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment-service  # Selects pods with this label
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP  # Internal service (DNS name: payment-service)
```

**Pod Definition:**

```yaml
# payment-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
      - name: payment-service
        image: payment-service:1.0.0
        ports:
        - containerPort: 8080
```

**Client Service:**

```java
// Order Service calls Payment Service
@Service
public class OrderService {
    @Autowired
    private RestTemplate restTemplate;
    
    public PaymentResult processPayment(Order order) {
        // Kubernetes DNS resolves payment-service
        // kube-proxy handles load balancing
        return restTemplate.postForObject(
            "http://payment-service/api/payments",  // Service name
            order,
            PaymentResult.class
        );
    }
}
```

**How it works:**
- Kubernetes DNS resolves `payment-service` to Service IP
- Service IP is virtual (handled by kube-proxy)
- kube-proxy routes to healthy pod endpoints
- Pods are selected by labels

### Netflix Implementation

Netflix uses Eureka for service discovery across 1000+ microservices:

1. **Eureka Clusters**: Multiple Eureka servers for high availability
2. **Zone Awareness**: Prefer instances in same availability zone
3. **Health Checks**: Custom health check endpoints
4. **Client-Side Load Balancing**: Ribbon for intelligent routing

### Uber Implementation

Uber uses Consul for service discovery:

1. **Multi-Datacenter**: Consul clusters per region, federated
2. **Health Checks**: HTTP, TCP, and script-based checks
3. **Service Mesh Integration**: Consul Connect for secure service-to-service communication
4. **Key-Value Store**: Also used for configuration

---

## 6️⃣ How to Implement or Apply It

### Step-by-Step: Setting Up Eureka Service Discovery

**Step 1: Create Eureka Server**

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```

```java
// EurekaServerApplication.java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# application.yml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
```

**Step 2: Configure Service as Eureka Client**

```xml
<!-- payment-service/pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```

```java
// PaymentServiceApplication.java
@SpringBootApplication
@EnableEurekaClient
public class PaymentServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentServiceApplication.class, args);
    }
}
```

```yaml
# application.yml
spring:
  application:
    name: payment-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

**Step 3: Configure Service Consumer**

```java
// OrderServiceApplication.java
@SpringBootApplication
@EnableEurekaClient
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

```java
// OrderService.java
@Service
public class OrderService {
    @Autowired
    private RestTemplate restTemplate;
    
    public PaymentResult processPayment(Order order) {
        return restTemplate.postForObject(
            "http://payment-service/api/payments",
            order,
            PaymentResult.class
        );
    }
}
```

**Step 4: Verify Registration**

Access Eureka dashboard: `http://localhost:8761`

You should see:
- payment-service instances
- order-service instances

### Using Feign Client (Declarative)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

@FeignClient(name = "payment-service")
public interface PaymentServiceClient {
    @PostMapping("/api/payments")
    PaymentResult processPayment(@RequestBody Order order);
    
    @GetMapping("/api/payments/{paymentId}")
    PaymentResult getPayment(@PathVariable String paymentId);
}
```

### Health Checks

```java
// Payment Service Health Check
@Component
public class PaymentServiceHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        // Check if service can process payments
        if (canProcessPayments()) {
            return Health.up()
                .withDetail("paymentGateway", "connected")
                .build();
        } else {
            return Health.down()
                .withDetail("paymentGateway", "disconnected")
                .build();
        }
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Tradeoffs: Client-Side vs Server-Side Discovery

**Client-Side Discovery (Eureka, Consul):**

**Advantages:**
- Direct communication (no extra hop)
- Client controls load balancing
- Can implement advanced routing (zone awareness, etc.)

**Disadvantages:**
- Requires client library per language
- Client complexity (caching, load balancing logic)
- Registry becomes single point of query

**Server-Side Discovery (Kubernetes):**

**Advantages:**
- Simple client (just DNS)
- Platform handles complexity
- Works with any language

**Disadvantages:**
- Extra hop through service proxy
- Less control over routing
- Platform-specific

### Common Pitfalls

**Pitfall 1: Stale Caches**

```java
// BAD: Cache never expires
private List<ServiceInstance> cachedInstances;
public ServiceInstance getInstance() {
    if (cachedInstances == null) {
        cachedInstances = discoveryClient.getInstances("payment-service");
    }
    return cachedInstances.get(0);  // Always uses first instance
}
```

**Solution**: Implement cache expiration and refresh.

**Pitfall 2: No Health Checks**

Service registers but is unhealthy. Registry still routes traffic to it.

**Solution**: Implement health checks. Eureka uses health endpoint, mark unhealthy instances as DOWN.

**Pitfall 3: Registry Single Point of Failure**

If Eureka server goes down, new discoveries fail.

**Solution**: Run Eureka in cluster mode (peer replication).

**Pitfall 4: Service Name Mismatches**

```java
// Order Service
restTemplate.getForObject("http://payment-service/...", ...);

// Payment Service registered as "paymentService" (camelCase)
// DNS/Registry lookup fails
```

**Solution**: Standardize service names (kebab-case recommended).

**Pitfall 5: Ignoring Network Partitions**

In network partition, service thinks registry is down, stops registering.

**Solution**: Implement circuit breakers, fallback mechanisms.

### Performance Considerations

**Registry Query Frequency:**

Querying registry on every request is slow. Use caching with TTL:

```java
@Cacheable(value = "serviceInstances", ttl = 30)
public List<ServiceInstance> getInstances(String serviceName) {
    return discoveryClient.getInstances(serviceName);
}
```

**Registry Load:**

With 1000 services × 10 instances = 10,000 registrations. Registry must scale.

**Solution**: Registry clustering, read replicas.

---

## 8️⃣ When NOT to Use This

### Anti-Patterns and Misuse Cases

**Don't use service discovery for:**

**1. External Services**

External APIs (payment gateways, third-party APIs) don't register with your registry. Use API Gateway or direct URLs.

**2. Static Services**

Services that never change (legacy systems, external databases). Hardcode or use DNS.

**3. Serverless Functions**

Lambda functions don't need service discovery (API Gateway routes to them).

### Situations Where This is Overkill

**Small Number of Services:**

With 2-3 services, hardcoded URLs might be simpler. Service discovery adds complexity.

**Single Instance Per Service:**

If each service runs on one instance, service discovery provides little value.

### Better Alternatives for Specific Scenarios

**API Gateway Pattern:**

For external-facing services, use API Gateway that handles discovery internally. Clients call gateway, gateway routes to services.

**Service Mesh:**

For advanced requirements (mTLS, observability, traffic management), service mesh handles discovery automatically.

---

## 9️⃣ Comparison with Alternatives

### Eureka vs Consul vs Kubernetes DNS

| Feature | Eureka | Consul | Kubernetes DNS |
|---------|--------|--------|----------------|
| **Pattern** | Client-side | Client-side/Server-side | Server-side |
| **Health Checks** | HTTP endpoint | Multiple types | Liveness/Readiness probes |
| **Language Support** | Java (Spring Cloud) | Multi-language | Any (DNS) |
| **Configuration** | Key-value store | Key-value store | ConfigMaps |
| **Service Mesh** | No | Consul Connect | Istio/Linkerd |
| **Use Case** | Spring Cloud apps | Multi-language, multi-DC | Kubernetes |

### Service Discovery vs API Gateway

**Service Discovery:**
- Service-to-service communication
- Internal network
- Direct service calls

**API Gateway:**
- Client-to-service communication
- External/internal
- Single entry point

**Use both**: API Gateway for external, Service Discovery for internal.

---

## 🔟 Interview Follow-up Questions WITH Answers

### L4 Level Questions

**Q1: What is service discovery in microservices?**

**Answer**: Service discovery is a mechanism that allows microservices to find and communicate with each other dynamically. Instead of hardcoding service URLs, services register themselves with a service registry, and clients query the registry to discover service instances. This is essential in microservices because instances are created/destroyed frequently, IP addresses change, and services need to scale dynamically.

**Q2: What's the difference between client-side and server-side service discovery?**

**Answer**:
- **Client-side**: Client queries registry, gets list of instances, picks one and calls directly. Client does load balancing. Example: Eureka + Ribbon. Advantage: Direct communication, client control. Disadvantage: Requires client library.

- **Server-side**: Client calls stable endpoint (DNS name), platform (load balancer/service proxy) queries registry and routes. Example: Kubernetes Services. Advantage: Simple client, works with any language. Disadvantage: Extra hop.

### L5 Level Questions

**Q3: How do you handle service discovery in a multi-region microservices deployment?**

**Answer**: 
1. **Registry per region**: Each region has its own service registry cluster (Eureka cluster per region)
2. **Federation**: Registries sync across regions (Consul federation, Eureka zones)
3. **Zone awareness**: Prefer instances in same region/zone for lower latency
4. **Failover**: If local registry unavailable, fallback to remote region
5. **Service mesh**: Service mesh (Istio) handles multi-region discovery automatically

Example: Eureka zones where each zone has its own Eureka cluster, and clients prefer same-zone instances.

**Q4: What happens when a service instance fails in a service discovery setup?**

**Answer**:
1. **Health check fails**: Instance stops responding to health checks
2. **Registry removes**: After timeout (e.g., 90 seconds for Eureka), registry marks instance as DOWN and removes from registry
3. **Client cache refresh**: Clients refresh their cache (typically every 30 seconds)
4. **Traffic stops**: Dead instance no longer receives traffic
5. **New instance**: New instance can start and register, automatically receiving traffic

**Q5: How does service discovery work with blue-green deployments?**

**Answer**:
1. **Blue deployment**: Current version (blue) registered in service registry
2. **Green deployment**: New version (green) deployed, registers with same service name
3. **Traffic split**: Service discovery returns both blue and green instances
4. **Gradual migration**: Load balancer routes percentage of traffic to green (canary)
5. **Blue removal**: After verification, blue instances deregister and shut down
6. **Green only**: All traffic goes to green

Service discovery automatically handles the transition - clients don't need code changes.

### L6 Level Questions

**Q6: You're designing service discovery for a microservices system with 1000 services. How do you ensure the registry doesn't become a bottleneck?**

**Answer**:
1. **Registry Clustering**: Run registry in cluster mode (Eureka cluster, Consul cluster) for high availability and load distribution
2. **Client-side Caching**: Clients cache service instances, refresh periodically (e.g., every 30 seconds) rather than querying on every request
3. **Read Replicas**: Use read replicas for discovery queries, write to primary for registration
4. **Zone/Region Partitioning**: Partition registry by zone/region, clients query local registry
5. **Service Mesh**: Use service mesh (Istio) which distributes service information to sidecars, reducing central registry load
6. **DNS-based**: For Kubernetes, use DNS which is highly scalable and cached
7. **Rate Limiting**: Implement rate limiting on registry to prevent abuse
8. **Monitoring**: Monitor registry load, scale horizontally as needed

**Q7: How do you handle service discovery during network partitions (split-brain scenario)?**

**Answer**:
Network partition splits cluster into two groups that can't communicate:

1. **Registry Partitioning**: Each partition has its own view of services
   - Partition A: Services 1-500 visible
   - Partition B: Services 501-1000 visible

2. **Client Behavior**:
   - Clients in Partition A can only discover services in Partition A
   - Clients in Partition B can only discover services in Partition B
   - Cross-partition calls fail

3. **Mitigation Strategies**:
   - **Last Known Good**: Clients cache last known instances, use cache if registry unreachable
   - **Circuit Breakers**: Fail fast if registry unavailable, prevent cascade failures
   - **Fallback Services**: Maintain static list of critical services as fallback
   - **Multi-Region**: Design for eventual consistency, accept temporary unavailability

4. **Prevention**:
   - Registry clustering across availability zones
   - Network redundancy
   - Health checks that detect partitions

5. **Recovery**: When partition heals, registries sync, clients refresh, normal operation resumes.

**Q8: Compare service discovery approaches for a microservices migration from monolith. What factors would influence your choice?**

**Answer**: 

Factors to consider:
- **Current Infrastructure**: Kubernetes → Use Kubernetes Services. AWS → Use AWS Cloud Map or ECS Service Discovery.
- **Team Expertise**: Java/Spring team → Eureka. Multi-language → Consul.
- **Scale**: < 100 services → Simpler solution. > 1000 services → Need clustering, caching.
- **Network Complexity**: Single region → Simple. Multi-region → Need federation.
- **Future Plans**: Planning service mesh → Choose compatible solution (Consul Connect, Istio).

**Recommendation for Migration:**

1. **Phase 1 (Initial)**: API Gateway pattern
   - Monolith exposes APIs
   - New microservices register with gateway
   - Gateway routes to services
   - Minimal changes to monolith

2. **Phase 2 (Growing)**: Client-side discovery (Eureka/Consul)
   - Services discover each other
   - Remove gateway dependency for service-to-service
   - More control, better performance

3. **Phase 3 (Mature)**: Service Mesh (Istio)
   - Advanced features (mTLS, observability, traffic management)
   - Discovery handled by mesh
   - Less application code

Migration path allows incremental adoption, reduces risk.

---

## 1️⃣1️⃣ One Clean Mental Summary

Service discovery in microservices is like a dynamic phone book that updates in real-time. Services register themselves with a registry (like restaurants registering locations), the registry tracks all instances and their health (like tracking which locations are open), and clients query the registry to find services (like calling a reservation system). Two main patterns: client-side discovery (client queries registry and calls directly, like Eureka) and server-side discovery (client calls stable endpoint, platform routes, like Kubernetes Services). Service discovery is essential because microservice instances are created/destroyed frequently, IP addresses change, and services need to scale dynamically. Without it, you'd need to hardcode hundreds of URLs and update them constantly. With it, services automatically find each other, dead instances are removed, and new instances are discovered automatically. Choose based on your infrastructure (Kubernetes → DNS, Spring Cloud → Eureka, multi-language → Consul) and requirements (simple → DNS, advanced → Service Mesh).






