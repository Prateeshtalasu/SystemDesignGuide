# Strangler Pattern

## 0️⃣ Prerequisites

Before understanding the Strangler Pattern, you should know:

- **Monolith vs Microservices**: The differences, tradeoffs, and when to use each (covered in topic 1)
- **API Gateway**: How API gateways route requests (covered in topic 14)
- **Service decomposition**: How to break down a monolith into services (covered in topics 1-3)
- **Domain-Driven Design**: Bounded contexts and how to identify service boundaries (covered in topic 3)
- **Load balancing and routing**: How requests are routed to different services

If you're not familiar with these, review them first. The Strangler Pattern is a migration strategy that builds on these concepts.

---

## 1️⃣ What problem does this exist to solve?

### The Pain Points

**Problem 1: Big Bang Migration is Risky**
- Rewrite entire monolith to microservices
- Months or years of development
- High risk: Everything breaks at once
- Can't deploy incrementally
- Business stops during migration

**Problem 2: Monolith Can't Be Replaced Overnight**
- Monolith is too large to rewrite quickly
- Business can't pause for migration
- Need to keep system running during migration
- Can't afford downtime

**Problem 3: Unknown Dependencies**
- Monolith has hidden dependencies
- Don't know all the code paths
- Can't identify all edge cases upfront
- Rewriting risks missing critical functionality

**Problem 4: Team Can't Work in Parallel**
- Big bang rewrite: One team, one codebase
- Can't parallelize development
- Slow progress
- High coordination overhead

**Problem 5: Can't Validate Incrementally**
- Can't test new architecture until everything is done
- Can't get user feedback early
- Can't measure improvements incrementally
- All-or-nothing approach

**Real-World Example: E-commerce Monolith Migration**

**Without Strangler Pattern (Big Bang):**
```
Year 1-2: Team rewrites entire monolith
  - No new features
  - Business waits
  - High risk
  - Can't test until done

Year 2: Cutover day
  - Shut down monolith
  - Turn on microservices
  - Everything breaks
  - Rollback to monolith
  - Migration fails
```

**What breaks without Strangler Pattern:**
- High risk of failure (all-or-nothing)
- Long development cycles (can't ship incrementally)
- Business disruption (can't add features during migration)
- Unknown issues surface only at cutover
- Can't validate approach until too late

---

## 2️⃣ Intuition and mental model

### The Strangler Fig Analogy

The pattern is named after the strangler fig tree:

**In Nature:**
- Strangler fig starts as a seed on a host tree
- Grows around the host tree gradually
- Over time, the fig tree becomes stronger
- Eventually, the host tree dies, but the fig tree remains
- The migration happens incrementally, not all at once

**In Software:**
- Start with existing monolith (host tree)
- Build new microservices (fig tree) around it
- Gradually route more traffic to microservices
- Over time, microservices handle more functionality
- Eventually, monolith is "strangled" (replaced)
- Migration happens incrementally, safely

### The Bridge Analogy

Think of Strangler Pattern like building a new bridge while the old one is still in use:

**Old Bridge (Monolith):**
- Still carries all traffic
- Can't shut it down
- Business depends on it

**New Bridge (Microservices):**
- Built lane by lane
- Each lane tested before opening
- Gradually shift traffic from old to new
- Old bridge remains until new is complete
- Zero downtime migration

**Key Insight:**
- Don't destroy old system until new is ready
- Migrate incrementally (one feature at a time)
- Test each piece before moving to next
- Can rollback at any point

---

## 3️⃣ How it works internally

### Architecture Overview

```
┌─────────────┐
│   Clients   │
└──────┬──────┘
       │
       │ All requests
       │
┌──────▼──────────────────┐
│   API Gateway / Router  │
│  (Strangler Facade)     │
└──────┬──────────────────┘
       │
       ├─────────────────┐
       │                 │
       │                 │
┌──────▼──────┐   ┌──────▼──────────┐
│  Monolith   │   │ Microservices   │
│  (Old)      │   │ (New)           │
│             │   │                 │
│  - Feature A│   │ - Feature B ✓   │
│  - Feature B│   │ - Feature C ✓   │
│  - Feature C│   │ - Feature D ✓   │
│  - Feature D│   │                 │
└─────────────┘   └──────────────────┘
```

### Core Components

**1. Strangler Facade (Router/Gateway)**
- Single entry point for all requests
- Routes requests to monolith or microservices
- Decision logic: "Is this feature migrated?"
- Can route based on feature flags, percentages, or user segments

**2. Monolith (Legacy System)**
- Original system, still running
- Handles non-migrated features
- Gradually loses functionality as features migrate
- Eventually becomes empty or is decommissioned

**3. Microservices (New System)**
- New services built incrementally
- Each service replaces part of monolith
- Services are independent and deployable
- Can be developed in parallel

**4. Feature Flags / Routing Rules**
- Control which requests go where
- Can route 0% → 10% → 50% → 100% gradually
- Can route by user segment (beta users first)
- Can rollback instantly

### Migration Flow

**Phase 1: Identify Feature to Migrate**
```
Monolith has:
  - User Management
  - Order Processing
  - Payment Processing
  - Inventory Management
  - Shipping

Choose: Order Processing (well-defined, independent)
```

**Phase 2: Build Microservice**
```
Build: Order Service
  - Extracts order logic from monolith
  - Implements same API contract
  - Deploys alongside monolith
  - Initially handles 0% of traffic
```

**Phase 3: Route Traffic Gradually**
```
Week 1: Route 0% to Order Service (monolith handles all)
Week 2: Route 5% to Order Service (test with small group)
Week 3: Route 25% to Order Service (expand)
Week 4: Route 50% to Order Service (half traffic)
Week 5: Route 100% to Order Service (fully migrated)
```

**Phase 4: Remove from Monolith**
```
Once 100% traffic on Order Service:
  - Remove order code from monolith
  - Monolith no longer handles orders
  - Order Service is fully independent
```

**Phase 5: Repeat for Next Feature**
```
Next: Payment Processing
  - Build Payment Service
  - Route gradually
  - Remove from monolith
  - Continue...
```

### Routing Strategies

**1. Percentage-Based Routing**
```java
// Route 10% of requests to new service
if (random() < 0.10) {
    routeToMicroservice();
} else {
    routeToMonolith();
}
```

**2. Feature Flag Routing**
```java
// Route based on feature flag
if (featureFlag.isEnabled("new-order-service", userId)) {
    routeToOrderService();
} else {
    routeToMonolith();
}
```

**3. User Segment Routing**
```java
// Route beta users to new service
if (user.isBetaUser()) {
    routeToMicroservice();
} else {
    routeToMonolith();
}
```

**4. Path-Based Routing**
```java
// Route specific endpoints
if (request.getPath().startsWith("/api/v2/orders")) {
    routeToOrderService();
} else {
    routeToMonolith();
}
```

---

## 4️⃣ Simulation-first explanation

### Simplest Possible Migration

Let's migrate a simple feature: **User Profile** from monolith to microservice.

**Setup:**
- Monolith (Spring Boot, handles everything)
- User Service (new microservice, handles user profiles)
- API Gateway (routes requests)

**Step-by-Step Flow:**

**1. Initial State (100% Monolith)**
```java
// Monolith: UserController.java
@RestController
public class UserController {
    
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable String id) {
        return userService.getUser(id);
    }
}

// API Gateway routes everything to monolith
@RestController
public class GatewayController {
    
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable String id) {
        // Route to monolith
        return restTemplate.getForObject(
            "http://monolith:8080/users/" + id, 
            User.class
        );
    }
}
```

**2. Build User Service (0% traffic)**
```java
// User Service: UserController.java
@RestController
public class UserController {
    
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable String id) {
        return userService.getUser(id);
    }
}

// Deploy alongside monolith
// Initially handles 0% of traffic
```

**3. Update Gateway with Routing Logic**
```java
// Gateway: Smart routing
@RestController
public class GatewayController {
    
    @Value("${user.service.traffic.percentage:0}")
    private int userServiceTrafficPercentage;
    
    @Autowired
    private RestTemplate restTemplate;
    
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable String id) {
        // Route based on percentage
        if (shouldRouteToUserService(id)) {
            return restTemplate.getForObject(
                "http://user-service:8081/users/" + id,
                User.class
            );
        } else {
            return restTemplate.getForObject(
                "http://monolith:8080/users/" + id,
                User.class
            );
        }
    }
    
    private boolean shouldRouteToUserService(String userId) {
        // Simple hash-based routing for consistent user experience
        int hash = userId.hashCode();
        int bucket = Math.abs(hash % 100);
        return bucket < userServiceTrafficPercentage;
    }
}
```

**4. Gradual Migration (0% → 100%)**

**Week 1: 0% (Testing)**
```yaml
# application.yml
user:
  service:
    traffic:
      percentage: 0  # All traffic to monolith
```

**Week 2: 10% (Small test)**
```yaml
user:
  service:
    traffic:
      percentage: 10  # 10% to user service, 90% to monolith
```

**Week 3: 50% (Half traffic)**
```yaml
user:
  service:
    traffic:
      percentage: 50  # 50% to user service, 50% to monolith
```

**Week 4: 100% (Fully migrated)**
```yaml
user:
  service:
    traffic:
      percentage: 100  # All traffic to user service
```

**5. Remove from Monolith**
```java
// Monolith: Remove user profile code
// UserController.java - DELETED
// UserService.java - DELETED
// UserRepository.java - DELETED

// Monolith no longer handles user profiles
// All traffic goes to User Service
```

**6. Monitor and Validate**
```java
// Gateway: Compare responses
@GetMapping("/users/{id}")
public User getUser(@PathVariable String id) {
    if (shouldRouteToUserService(id)) {
        User user = userService.getUser(id);
        // Log for monitoring
        log.info("User {} served by User Service", id);
        return user;
    } else {
        User user = monolith.getUser(id);
        log.info("User {} served by Monolith", id);
        return user;
    }
}
```

### Adding More Features

**Migrate Order Service:**
```java
// Gateway: Add order routing
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable String id) {
    if (shouldRouteToOrderService(id)) {
        return orderService.getOrder(id);
    } else {
        return monolith.getOrder(id);
    }
}

// Gradually increase order service traffic
// 0% → 10% → 50% → 100%
// Remove from monolith when 100%
```

**Result:**
- User Service: 100% migrated ✓
- Order Service: 100% migrated ✓
- Payment Service: 50% migrated (in progress)
- Inventory Service: 0% migrated (not started)
- Monolith: Getting smaller each week

---

## 5️⃣ How engineers actually use this in production

### Real-World Implementations

**Amazon: Strangler Pattern for E-commerce**
- Started with monolith in 1990s
- Gradually extracted services: Product Catalog, Order Management, Payment, Shipping
- Used API Gateway to route traffic
- Migration took years, zero downtime
- Now fully microservices

**Netflix: Migration from Monolith to Microservices**
- Started with monolith handling video streaming
- Extracted services: User Service, Recommendation Service, Playback Service
- Used feature flags for gradual rollout
- Each service migrated independently
- Now handles billions of requests per day

**Uber: Strangler Pattern for Ride-Sharing**
- Started with monolith
- Extracted: Trip Service, Payment Service, Driver Service, Rider Service
- Used service mesh for routing
- Gradual migration over 2-3 years
- Zero downtime during migration

**Etsy: Incremental Migration**
- Monolith handling marketplace
- Extracted: Search Service, Listing Service, Payment Service
- Used percentage-based routing
- Migrated one feature at a time
- Validated each step before proceeding

### Production Patterns

**1. Database Strangulation**
```
Phase 1: Monolith writes to old DB
Phase 2: Monolith writes to old DB, new service reads from new DB
Phase 3: Monolith writes to both DBs (dual write)
Phase 4: New service writes to new DB, monolith reads from old DB
Phase 5: Everything uses new DB, old DB decommissioned
```

**2. API Versioning Strategy**
```
/api/v1/users → Monolith (old)
/api/v2/users → User Service (new)

Gradually migrate clients from v1 to v2
When all clients on v2, remove v1
```

**3. Feature Flag Based Routing**
```java
// Use feature flags for gradual rollout
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable String id) {
    if (featureFlagService.isEnabled("new-order-service", userId)) {
        return orderService.getOrder(id);
    } else {
        return monolith.getOrder(id);
    }
}

// Rollout plan:
// Day 1: Enable for internal users (0.1%)
// Day 7: Enable for beta users (5%)
// Day 14: Enable for all users (100%)
```

**4. Canary Deployment**
```java
// Route new version to canary instances
if (isCanaryUser(userId)) {
    return newService.getOrder(id);
} else {
    return monolith.getOrder(id);
}

// Monitor canary metrics
// If metrics good, expand canary
// If metrics bad, rollback
```

**5. Shadow Mode**
```java
// Send requests to both, compare results
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable String id) {
    Order monolithOrder = monolith.getOrder(id);
    
    // Also call new service (shadow mode)
    Order newServiceOrder = newService.getOrder(id);
    
    // Compare (log differences)
    compareOrders(monolithOrder, newServiceOrder);
    
    // Return monolith result (new service not serving traffic yet)
    return monolithOrder;
}
```

### Production Tooling

**Monitoring:**
- Traffic split: % to monolith vs microservices
- Error rates: Compare monolith vs microservices
- Latency: Compare response times
- Feature migration progress: Which features migrated

**Observability:**
- Distributed tracing: Follow requests through monolith vs microservices
- Logging: Tag requests with source (monolith vs service)
- Metrics: Track migration progress

**Rollback Strategy:**
- Feature flags: Instant rollback (disable flag)
- Percentage routing: Reduce to 0% instantly
- Database: Keep old DB until migration complete

---

## 6️⃣ How to implement or apply it

### Spring Boot Implementation

**Step 1: Create API Gateway (Strangler Facade)**
```java
// GatewayApplication.java
@SpringBootApplication
@EnableZuulProxy  // Or use Spring Cloud Gateway
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

**Step 2: Create Routing Service**
```java
// RoutingService.java
@Service
public class RoutingService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Value("${order.service.traffic.percentage:0}")
    private int orderServiceTrafficPercentage;
    
    @Value("${user.service.traffic.percentage:0}")
    private int userServiceTrafficPercentage;
    
    private final String monolithUrl = "http://monolith:8080";
    private final String orderServiceUrl = "http://order-service:8081";
    private final String userServiceUrl = "http://user-service:8082";
    
    public <T> T routeRequest(String path, Class<T> responseType, String userId) {
        // Determine routing based on path and traffic percentage
        if (path.startsWith("/orders")) {
            return routeOrderRequest(path, responseType, userId);
        } else if (path.startsWith("/users")) {
            return routeUserRequest(path, responseType, userId);
        } else {
            // Not migrated, route to monolith
            return restTemplate.getForObject(
                monolithUrl + path, 
                responseType
            );
        }
    }
    
    private <T> T routeOrderRequest(String path, Class<T> responseType, String userId) {
        if (shouldRouteToService(userId, orderServiceTrafficPercentage)) {
            log.info("Routing {} to Order Service", path);
            return restTemplate.getForObject(
                orderServiceUrl + path, 
                responseType
            );
        } else {
            log.info("Routing {} to Monolith", path);
            return restTemplate.getForObject(
                monolithUrl + path, 
                responseType
            );
        }
    }
    
    private <T> T routeUserRequest(String path, Class<T> responseType, String userId) {
        if (shouldRouteToService(userId, userServiceTrafficPercentage)) {
            log.info("Routing {} to User Service", path);
            return restTemplate.getForObject(
                userServiceUrl + path, 
                responseType
            );
        } else {
            log.info("Routing {} to Monolith", path);
            return restTemplate.getForObject(
                monolithUrl + path, 
                responseType
            );
        }
    }
    
    private boolean shouldRouteToService(String userId, int percentage) {
        if (percentage == 0) return false;
        if (percentage == 100) return true;
        
        // Consistent hashing: same user always routes to same service
        int hash = userId.hashCode();
        int bucket = Math.abs(hash % 100);
        return bucket < percentage;
    }
}
```

**Step 3: Create Gateway Controller**
```java
// GatewayController.java
@RestController
@Slf4j
public class GatewayController {
    
    @Autowired
    private RoutingService routingService;
    
    @GetMapping("/orders/{id}")
    public Order getOrder(
            @PathVariable String id,
            @RequestHeader(value = "X-User-Id", required = false) String userId) {
        
        return routingService.routeRequest(
            "/orders/" + id,
            Order.class,
            userId != null ? userId : id
        );
    }
    
    @GetMapping("/users/{id}")
    public User getUser(
            @PathVariable String id,
            @RequestHeader(value = "X-User-Id", required = false) String userId) {
        
        return routingService.routeRequest(
            "/users/" + id,
            User.class,
            userId != null ? userId : id
        );
    }
    
    // Fallback for non-migrated endpoints
    @GetMapping("/**")
    public ResponseEntity<?> fallback(HttpServletRequest request) {
        String path = request.getRequestURI();
        log.info("Fallback: Routing {} to Monolith", path);
        
        // Route to monolith
        return restTemplate.getForEntity(
            "http://monolith:8080" + path,
            Object.class
        );
    }
}
```

**Step 4: Configuration (application.yml)**
```yaml
# Gateway configuration
server:
  port: 8080

# Traffic routing percentages
order:
  service:
    traffic:
      percentage: 0  # Start at 0%, increase gradually

user:
  service:
    traffic:
      percentage: 0  # Start at 0%, increase gradually

# Service URLs
services:
  monolith:
    url: http://monolith:8080
  order-service:
    url: http://order-service:8081
  user-service:
    url: http://user-service:8082
```

**Step 5: Feature Flag Integration**
```java
// FeatureFlagService.java
@Service
public class FeatureFlagService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    private final String featureFlagServiceUrl = "http://feature-flag-service:8083";
    
    public boolean isEnabled(String feature, String userId) {
        // Call feature flag service
        FeatureFlag flag = restTemplate.getForObject(
            featureFlagServiceUrl + "/flags/" + feature + "/users/" + userId,
            FeatureFlag.class
        );
        return flag != null && flag.isEnabled();
    }
}

// Update RoutingService to use feature flags
private boolean shouldRouteToService(String userId, int percentage, String feature) {
    // Check feature flag first
    if (featureFlagService.isEnabled(feature, userId)) {
        return true;
    }
    
    // Fall back to percentage-based routing
    return shouldRouteToService(userId, percentage);
}
```

**Step 6: Monitoring and Metrics**
```java
// MetricsService.java
@Service
public class MetricsService {
    
    private final MeterRegistry meterRegistry;
    
    public void recordRouting(String service, String path, long latency) {
        Counter.builder("routing.requests")
            .tag("service", service)
            .tag("path", path)
            .register(meterRegistry)
            .increment();
        
        Timer.builder("routing.latency")
            .tag("service", service)
            .tag("path", path)
            .register(meterRegistry)
            .record(latency, TimeUnit.MILLISECONDS);
    }
}

// Update RoutingService to record metrics
public <T> T routeRequest(String path, Class<T> responseType, String userId) {
    long startTime = System.currentTimeMillis();
    String service = determineService(path, userId);
    
    try {
        T result = doRoute(path, responseType, userId, service);
        long latency = System.currentTimeMillis() - startTime;
        metricsService.recordRouting(service, path, latency);
        return result;
    } catch (Exception e) {
        metricsService.recordError(service, path, e);
        throw e;
    }
}
```

**Step 7: Gradual Migration Script**
```bash
#!/bin/bash
# migrate.sh - Gradually increase traffic percentage

# Week 1: 0% (testing)
updateConfig "order.service.traffic.percentage=0"

# Week 2: 10%
updateConfig "order.service.traffic.percentage=10"
sleep 7d
monitorMetrics

# Week 3: 25%
updateConfig "order.service.traffic.percentage=25"
sleep 7d
monitorMetrics

# Week 4: 50%
updateConfig "order.service.traffic.percentage=50"
sleep 7d
monitorMetrics

# Week 5: 100%
updateConfig "order.service.traffic.percentage=100"
sleep 7d
monitorMetrics

# Remove from monolith
echo "Migration complete, remove order code from monolith"
```

---

## 7️⃣ Tradeoffs, pitfalls, and common mistakes

### Tradeoffs

**Pros:**
- ✅ Low risk: Migrate incrementally, can rollback
- ✅ Zero downtime: System keeps running during migration
- ✅ Parallel development: Teams work on different services
- ✅ Incremental validation: Test each piece before proceeding
- ✅ Business continuity: Can add features during migration
- ✅ Learn and adapt: Adjust approach based on early results

**Cons:**
- ❌ Longer timeline: Takes months/years vs weeks
- ❌ Temporary complexity: Running both systems simultaneously
- ❌ Data synchronization: May need to sync data between systems
- ❌ Testing overhead: Must test both paths
- ❌ Resource usage: Running monolith + microservices

### Common Pitfalls

**1. Not Removing Code from Monolith**
```java
// BAD: Keep old code in monolith "just in case"
// Monolith still has order code even though 100% traffic on Order Service
// Monolith becomes bloated, hard to maintain
```

**Solution:**
- Remove code from monolith once 100% migrated
- Keep monolith clean
- Use version control to recover if needed

**2. Inconsistent Data Between Systems**
```java
// BAD: Monolith and microservice have different data
// User updates profile in monolith
// User Service doesn't know about update
// Inconsistent state
```

**Solution:**
- Use database replication during migration
- Or use event-driven updates
- Or route all writes to one system, reads can go to both

**3. Routing Logic Too Complex**
```java
// BAD: Complex routing logic, hard to understand
if (user.isPremium() && order.isLarge() && time.isBusinessHours()) {
    // 50% to new service
} else if (user.isBeta() && order.isSmall()) {
    // 25% to new service
} // ... 20 more conditions
```

**Solution:**
- Keep routing simple
- Use percentage-based or feature flag routing
- Document routing rules clearly

**4. Not Monitoring Migration Progress**
```java
// BAD: No visibility into migration
// Don't know which features migrated
// Don't know error rates
// Can't make informed decisions
```

**Solution:**
- Track traffic percentages
- Monitor error rates per service
- Track migration progress dashboard
- Alert on anomalies

**5. Migrating Too Fast**
```java
// BAD: Jump from 0% to 100% in one day
// No time to validate
// Issues affect all users
```

**Solution:**
- Gradual rollout: 0% → 10% → 25% → 50% → 100%
- Wait days/weeks between increases
- Monitor at each step
- Rollback if issues

**6. Not Testing Both Paths**
```java
// BAD: Only test new service
// Don't test monolith path
// Monolith breaks, affects 50% of users
```

**Solution:**
- Test both monolith and microservice paths
- Use shadow mode to compare
- Load test both systems

**7. Database Migration Issues**
```java
// BAD: Migrate service but not database
// Service reads from new DB, monolith writes to old DB
// Data inconsistency
```

**Solution:**
- Plan database migration separately
- Use dual-write pattern during transition
- Sync data between databases
- Migrate database after service migration

### Performance Gotchas

**1. Double Database Queries**
```java
// BAD: Gateway calls both monolith and microservice
// Compares results, but doubles load
```

**Solution:**
- Only use shadow mode for testing
- Don't run in production
- Use feature flags to control

**2. Latency from Routing Logic**
```java
// BAD: Complex routing logic adds latency
// Every request checks feature flags, percentages, etc.
```

**Solution:**
- Cache routing decisions
- Keep routing logic simple
- Use fast data structures (hash maps)

**3. Service Discovery Overhead**
```java
// BAD: Gateway discovers services on every request
// Adds latency
```

**Solution:**
- Cache service locations
- Use service mesh for discovery
- Pre-warm connections

### Security Considerations

**1. Authentication in Both Systems**
- Monolith and microservices both need auth
- Duplicate auth logic
- Solution: Use API Gateway for auth, forward to services

**2. Data Access Control**
- Ensure same permissions in both systems
- Don't expose more data in new service
- Solution: Centralize authorization logic

---

## 8️⃣ When NOT to use this

### Anti-Patterns and Misuse

**1. Small System**
- System is small, can rewrite quickly
- Overhead not worth it
- Use: Big bang rewrite

**2. System is Beyond Repair**
- Monolith is too broken to keep running
- Better to start fresh
- Use: Greenfield development

**3. No Business Continuity Requirement**
- Can afford downtime
- Business can pause
- Use: Big bang migration

**4. Simple Migration**
- System is simple, few dependencies
- Can migrate quickly
- Use: Direct migration

**5. Team Lacks Discipline**
- Team can't handle gradual migration
- Will cut corners
- Use: Big bang with strict process

### Signs You've Chosen Wrong

**Red Flags:**
- Migration taking too long (years without progress)
- Both systems running forever (never removing monolith)
- Complex routing logic (hard to maintain)
- Data inconsistencies (not syncing properly)
- No monitoring (flying blind)
- Migrating too fast (jumping percentages)

---

## 9️⃣ Comparison with Alternatives

### Strangler Pattern vs Big Bang Rewrite

**Big Bang Rewrite:**
- Rewrite everything at once
- ✅ Faster (if successful)
- ✅ Clean slate
- ❌ High risk (all-or-nothing)
- ❌ Long development (no incremental value)
- ❌ Can't validate until done

**Strangler Pattern:**
- Migrate incrementally
- ✅ Low risk (can rollback)
- ✅ Incremental value
- ✅ Can validate early
- ❌ Longer timeline
- ❌ Temporary complexity

**When to Choose:**
- Big Bang: Small system, can afford risk, need fast migration
- Strangler: Large system, need business continuity, want low risk

### Strangler Pattern vs Branch by Abstraction

**Branch by Abstraction:**
- Create abstraction layer in monolith
- Implement new system behind abstraction
- Switch implementation gradually
- ✅ No separate services during migration
- ❌ Still in monolith codebase

**Strangler Pattern:**
- Build separate services
- Route traffic gradually
- ✅ Services are independent
- ✅ Can deploy independently
- ❌ More infrastructure

**When to Choose:**
- Branch by Abstraction: Want to stay in monolith codebase
- Strangler: Want independent services from start

### Strangler Pattern vs Database-First Migration

**Database-First Migration:**
- Migrate database first
- Then migrate services
- ✅ Data migration done early
- ❌ Services must work with new DB immediately

**Strangler Pattern:**
- Migrate services first
- Database migration separate
- ✅ Services can use existing DB
- ❌ Database migration later

**When to Choose:**
- Database-First: Database is main constraint
- Strangler: Services are main constraint

---

## 🔟 Interview follow-up questions WITH answers

### L4 (Junior) Questions

**Q1: What is the Strangler Pattern and why is it used?**
**A:** The Strangler Pattern is a migration strategy to gradually replace a monolith with microservices. Instead of rewriting everything at once (big bang), you build new microservices incrementally and gradually route traffic from the monolith to the new services. It's used because big bang migrations are risky (everything breaks at once), take too long (can't ship incrementally), and require business to pause. Strangler Pattern allows zero-downtime migration with low risk.

**Q2: How does traffic routing work in Strangler Pattern?**
**A:** An API Gateway (strangler facade) sits in front of both the monolith and microservices. It routes requests based on configuration: percentage-based (10% to new service, 90% to monolith), feature flags (beta users to new service), or path-based (specific endpoints to new service). As migration progresses, you increase the percentage (0% → 10% → 50% → 100%) until all traffic goes to the new service, then remove code from monolith.

**Q3: What happens to the monolith during migration?**
**A:** The monolith continues running and handling non-migrated features. As features migrate to microservices, the monolith gradually loses functionality. Once a feature is 100% migrated, you remove that code from the monolith. Eventually, the monolith becomes empty (or very small) and can be decommissioned. The key is that the monolith stays running throughout the migration, ensuring zero downtime.

### L5 (Mid-Level) Questions

**Q4: How do you handle data consistency during Strangler migration?**
**A:** 
- **Dual Write:** Write to both old and new databases during transition
- **Read from Both:** Compare results, log differences
- **Event-Driven Sync:** Publish events from monolith, new services consume and sync
- **Database Replication:** Replicate data from old DB to new DB
- **Accept Temporary Inconsistency:** For non-critical data, accept eventual consistency
- **Route Writes to One:** All writes go to one system, reads can go to both

**Q5: How do you decide which feature to migrate first?**
**A:**
- **Well-Defined Boundaries:** Features with clear domain boundaries (User, Order, Payment)
- **Independent:** Features with few dependencies on other parts
- **High Value:** Features that provide most value when migrated
- **Low Risk:** Features that are less critical (can afford issues)
- **Team Expertise:** Features the team understands well
- **Common Approach:** Start with read-heavy, well-isolated features (User Profile), then move to write-heavy, complex features (Order Processing)

**Q6: How do you monitor migration progress?**
**A:**
- **Traffic Metrics:** Track % of traffic to monolith vs microservices per feature
- **Error Rates:** Compare error rates between monolith and new services
- **Latency:** Compare response times
- **Migration Dashboard:** Visualize which features are migrated, progress percentage
- **Business Metrics:** Ensure business metrics (revenue, user satisfaction) don't degrade
- **Alerting:** Alert on anomalies (high error rates, latency spikes)

**Q7: What is shadow mode and when would you use it?**
**A:** Shadow mode is when you send requests to both the monolith and new service, but only return the monolith response. You compare the results to validate the new service works correctly without risking user experience. Use it during early migration (0-10% phase) to validate the new service before routing real traffic. Once validated, switch to actual routing. Don't use in production at scale (doubles load).

### L6 (Senior) Questions

**Q8: Design a Strangler Pattern migration for an e-commerce monolith with 10 major features.**
**A:**
- **Phase 1: Setup**
  - Deploy API Gateway (strangler facade)
  - Route 100% traffic through gateway to monolith
  - Set up monitoring and feature flags
  
- **Phase 2: Identify Migration Order**
  - User Service (read-heavy, well-isolated)
  - Product Catalog Service (read-heavy, independent)
  - Order Service (write-heavy, more complex)
  - Payment Service (critical, needs careful migration)
  - Shipping Service, Inventory Service, etc.
  
- **Phase 3: Migrate Each Feature**
  - Build microservice
  - Deploy alongside monolith (0% traffic)
  - Route 10% traffic, monitor for 1 week
  - Increase to 25%, monitor for 1 week
  - Increase to 50%, monitor for 1 week
  - Increase to 100%, monitor for 1 week
  - Remove code from monolith
  
- **Phase 4: Database Migration**
  - For each service, migrate database
  - Use dual-write pattern
  - Sync data
  - Switch reads to new DB
  - Switch writes to new DB
  - Decommission old DB
  
- **Phase 5: Decommission Monolith**
  - When all features migrated, decommission monolith
  - Timeline: 12-18 months
  - Zero downtime throughout

**Q9: How would you handle a situation where the new microservice performs worse than the monolith?**
**A:**
- **Immediate Rollback:** Reduce traffic percentage to 0% (route back to monolith)
- **Investigate:** Profile the new service, identify bottlenecks (database queries, network calls, inefficient code)
- **Fix Issues:** Optimize queries, add caching, fix performance bugs
- **Re-test:** Load test the fixed service
- **Gradual Re-rollout:** Start with 1% traffic, monitor closely, gradually increase
- **Compare Metrics:** Ensure new service matches or exceeds monolith performance before proceeding
- **Document Learnings:** Understand why performance was worse, prevent in future migrations

**Q10: How do you handle shared code and dependencies during Strangler migration?**
**A:**
- **Extract Shared Libraries:** Common utilities, models, interfaces → shared libraries
- **Version Libraries:** Version shared libraries, services use compatible versions
- **API Contracts:** Define clear API contracts between services
- **Event Contracts:** Use schema registry for events
- **Avoid Code Duplication:** Don't copy-paste code, extract to libraries
- **Gradual Extraction:** Extract shared code as you migrate features
- **Documentation:** Document shared dependencies and versions

**Q11: Compare Strangler Pattern with other migration strategies. When would you choose each?**
**A:**
- **Strangler Pattern:** Large system, need business continuity, want low risk, can take time. Best for: Production systems that can't have downtime.
- **Big Bang Rewrite:** Small system, can afford risk, need fast migration. Best for: Small systems, greenfield projects.
- **Branch by Abstraction:** Want to stay in monolith codebase, gradual refactoring. Best for: Refactoring within monolith.
- **Database-First:** Database is main constraint, need to migrate data first. Best for: Data migration is critical path.
- **Service Extraction:** Extract services one at a time, but keep in monolith repo. Best for: When you want services but not separate deployments yet.

---

## 1️⃣1️⃣ One clean mental summary

The Strangler Pattern is a migration strategy that gradually replaces a monolith with microservices by routing traffic incrementally from the old system to the new one. Instead of rewriting everything at once (risky big bang approach), you build new microservices one feature at a time and use an API Gateway to gradually shift traffic (0% → 10% → 50% → 100%). Once a feature is fully migrated, you remove it from the monolith. This allows zero-downtime migration with low risk, since you can rollback at any point and validate each step before proceeding. The tradeoff is a longer timeline (months to years), but you get incremental value, parallel development, and business continuity throughout the migration. Think of it like building a new bridge lane by lane while the old bridge still carries traffic, then gradually shifting cars to the new bridge until the old one is no longer needed.

