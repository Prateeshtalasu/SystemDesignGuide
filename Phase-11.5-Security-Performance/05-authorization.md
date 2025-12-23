# 🔐 Authorization: RBAC, ABAC, and Access Control Patterns

---

## 0️⃣ Prerequisites

Before diving into authorization, you should understand:

- **Authentication vs Authorization**: Authentication proves WHO you are (identity). Authorization determines WHAT you can do (permissions). You must authenticate before you can authorize.
- **JWT Tokens**: How tokens carry user identity and claims. Covered in `03-authentication-jwt.md`.
- **OAuth2 Scopes**: How OAuth2 limits access to specific resources. Covered in `04-oauth2-openid-connect.md`.

**Quick Refresher**: After authentication, you know the user is "Alice" (identity verified). Authorization answers: "Can Alice delete this document?", "Can Alice view salary data?", "Can Alice approve expenses over $10,000?"

---

## 1️⃣ What Problem Does Authorization Exist to Solve?

### The Core Problem: Not Everyone Should Access Everything

Authentication tells us WHO the user is. But that's not enough:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE AUTHORIZATION PROBLEM                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Scenario: Hospital System                                               │
│                                                                          │
│  Authenticated Users:                                                    │
│  • Dr. Smith (Cardiologist)                                             │
│  • Nurse Johnson (ER)                                                   │
│  • Bob (IT Support)                                                     │
│  • Alice (Patient)                                                       │
│                                                                          │
│  Question: Who can access Alice's medical records?                      │
│                                                                          │
│  Without Authorization:                                                  │
│  ❌ Everyone authenticated can access everything                        │
│  ❌ IT support could read medical records                               │
│  ❌ Any doctor could access any patient                                 │
│  ❌ Patients could modify their own records                             │
│                                                                          │
│  With Authorization:                                                     │
│  ✅ Only Alice's treating doctors can view her records                  │
│  ✅ Nurses can view but not modify                                      │
│  ✅ IT support has no access to medical data                            │
│  ✅ Alice can view her own records but not modify                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### What Breaks Without Proper Authorization

| Problem | Impact |
|---------|--------|
| No authorization | Any authenticated user accesses anything |
| Hardcoded permissions | Can't change without code deployment |
| Inconsistent checks | Some endpoints protected, others not |
| IDOR vulnerabilities | Users access other users' data by changing IDs |
| Privilege escalation | Users gain admin access |

### Real-World Authorization Failures

**2019 Capital One Breach**: A misconfigured firewall allowed an authenticated user (the attacker) to access resources they shouldn't have. Proper authorization would have limited access even after authentication bypass.

**Facebook Cambridge Analytica**: Users authorized an app to access their data, but the app also accessed their friends' data without proper authorization checks.

---

## 2️⃣ Intuition and Mental Model

### The Building Security Analogy

Think of authorization like building security:

**Authentication** = Badge that proves you work here
**Authorization** = Which doors your badge opens

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BUILDING SECURITY ANALOGY                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  RBAC (Role-Based):                                                      │
│  "Employees can access floors 1-5, Managers can access floor 6"         │
│  Your badge says "Manager" → You can access floors 1-6                  │
│                                                                          │
│  ABAC (Attribute-Based):                                                 │
│  "Access if: department=Engineering AND clearance>=SECRET                │
│              AND time between 9am-6pm AND location=HQ"                  │
│  Multiple attributes checked, not just role                             │
│                                                                          │
│  ReBAC (Relationship-Based):                                            │
│  "You can access files you own or files shared with you"                │
│  Access based on your relationship to the resource                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ Authorization Models

### RBAC (Role-Based Access Control)

The most common model. Users are assigned roles, roles have permissions.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RBAC MODEL                                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  USERS              ROLES                 PERMISSIONS                   │
│  ┌───────┐         ┌───────────┐         ┌────────────────────┐        │
│  │ Alice │────────>│   ADMIN   │────────>│ users:create       │        │
│  └───────┘         └───────────┘         │ users:read         │        │
│                          │               │ users:update       │        │
│  ┌───────┐               │               │ users:delete       │        │
│  │  Bob  │───┐           │               │ orders:*           │        │
│  └───────┘   │           │               │ reports:*          │        │
│              │           │               └────────────────────┘        │
│  ┌───────┐   │     ┌───────────┐         ┌────────────────────┐        │
│  │ Carol │───┴────>│  MANAGER  │────────>│ users:read         │        │
│  └───────┘         └───────────┘         │ orders:*           │        │
│                          │               │ reports:read       │        │
│  ┌───────┐               │               └────────────────────┘        │
│  │ David │───┐           │                                              │
│  └───────┘   │           │                                              │
│              │     ┌───────────┐         ┌────────────────────┐        │
│  ┌───────┐   │     │   USER    │────────>│ orders:read (own)  │        │
│  │  Eve  │───┴────>│           │         │ orders:create      │        │
│  └───────┘         └───────────┘         └────────────────────┘        │
│                                                                          │
│  Hierarchy: ADMIN > MANAGER > USER                                      │
│  Higher roles inherit lower role permissions                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**RBAC Pros**:
- Simple to understand and implement
- Easy to audit (who has what role)
- Works well for organizational hierarchies

**RBAC Cons**:
- Role explosion (too many specific roles)
- Can't express complex conditions
- Static (doesn't consider context)

### ABAC (Attribute-Based Access Control)

More flexible. Decisions based on attributes of user, resource, action, and environment.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ABAC MODEL                                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  POLICY: "Allow if all conditions are true"                             │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  SUBJECT ATTRIBUTES        │  RESOURCE ATTRIBUTES               │    │
│  │  • user.department = "HR"  │  • document.classification = "HR"  │    │
│  │  • user.clearance >= 3     │  • document.owner                  │    │
│  │  • user.location           │  • document.sensitivity            │    │
│  │  • user.title              │  • document.department             │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  ACTION ATTRIBUTES         │  ENVIRONMENT ATTRIBUTES            │    │
│  │  • action.type = "read"    │  • env.time (9am-6pm)              │    │
│  │  • action.type = "write"   │  • env.ip_address (internal)       │    │
│  │  • action.type = "delete"  │  • env.device_type (managed)       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Example Policy:                                                         │
│  ALLOW read document WHERE                                               │
│    subject.department == resource.department AND                        │
│    subject.clearance >= resource.sensitivity AND                        │
│    environment.time BETWEEN "09:00" AND "18:00" AND                     │
│    environment.network == "corporate"                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**ABAC Pros**:
- Very flexible and expressive
- Can handle complex business rules
- Context-aware (time, location, etc.)

**ABAC Cons**:
- More complex to implement and maintain
- Harder to audit and understand
- Policy management can become unwieldy

### ReBAC (Relationship-Based Access Control)

Access based on relationships between users and resources. Think Google Docs sharing.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ReBAC MODEL                                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  RELATIONSHIPS:                                                          │
│                                                                          │
│  ┌──────────┐      owner       ┌────────────┐                          │
│  │  Alice   │─────────────────>│ Document A │                          │
│  └──────────┘                  └────────────┘                          │
│       │                              ▲                                   │
│       │ member                       │ parent                           │
│       ▼                              │                                   │
│  ┌──────────┐      viewer      ┌────────────┐                          │
│  │ Team X   │─────────────────>│  Folder B  │                          │
│  └──────────┘                  └────────────┘                          │
│       ▲                                                                  │
│       │ member                                                           │
│       │                                                                  │
│  ┌──────────┐                                                           │
│  │   Bob    │                                                           │
│  └──────────┘                                                           │
│                                                                          │
│  ACCESS RULES:                                                           │
│  • owner can read, write, delete, share                                 │
│  • editor can read, write                                               │
│  • viewer can read                                                       │
│  • member of a group inherits group's permissions                       │
│  • permissions on folder apply to contained documents                   │
│                                                                          │
│  Can Bob read Document A?                                                │
│  Bob → member of → Team X → viewer of → Folder B → parent of → Doc A   │
│  YES! Bob can read Document A through inherited permissions             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**ReBAC Pros**:
- Natural for document/resource sharing
- Handles inheritance elegantly
- Scales to complex permission structures

**ReBAC Cons**:
- Requires graph database or specialized system
- Permission checks can be expensive (graph traversal)
- Complex to implement from scratch

---

## 4️⃣ How Authorization Works Internally

### Policy Decision Point (PDP) Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AUTHORIZATION ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                         APPLICATION                                │  │
│  │  ┌─────────────────────────────────────────────────────────────┐  │  │
│  │  │                  Policy Enforcement Point (PEP)              │  │  │
│  │  │  • Intercepts requests                                       │  │  │
│  │  │  • Asks PDP for decision                                     │  │  │
│  │  │  • Enforces decision (allow/deny)                           │  │  │
│  │  └──────────────────────────┬──────────────────────────────────┘  │  │
│  └─────────────────────────────┼─────────────────────────────────────┘  │
│                                │                                         │
│                                │ Authorization Request                   │
│                                │ {subject, action, resource, context}    │
│                                ▼                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                  Policy Decision Point (PDP)                       │  │
│  │  ┌─────────────────────────────────────────────────────────────┐  │  │
│  │  │                    Policy Engine                             │  │  │
│  │  │  • Evaluates policies against request                       │  │  │
│  │  │  • Returns ALLOW, DENY, or NOT_APPLICABLE                   │  │  │
│  │  └──────────────────────────┬──────────────────────────────────┘  │  │
│  │                             │                                      │  │
│  │  ┌──────────────────────────┴──────────────────────────────────┐  │  │
│  │  │                 Policy Information Point (PIP)               │  │  │
│  │  │  • Fetches additional attributes                            │  │  │
│  │  │  • User attributes from directory                           │  │  │
│  │  │  • Resource attributes from database                        │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  │                                                                    │  │
│  │  ┌─────────────────────────────────────────────────────────────┐  │  │
│  │  │              Policy Administration Point (PAP)               │  │  │
│  │  │  • Manages policies                                         │  │  │
│  │  │  • Policy CRUD operations                                   │  │  │
│  │  │  • Policy versioning                                        │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Centralized vs Distributed Authorization

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CENTRALIZED AUTHORIZATION                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                          │
│  │Service A │    │Service B │    │Service C │                          │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘                          │
│       │               │               │                                  │
│       └───────────────┼───────────────┘                                 │
│                       │                                                  │
│                       ▼                                                  │
│              ┌────────────────┐                                         │
│              │  Central PDP   │  ← Single source of truth               │
│              │  (OPA, etc.)   │  ← Consistent policies                  │
│              └────────────────┘  ← Potential bottleneck                 │
│                                                                          │
│  Pros: Consistent, easy to audit, single policy update                  │
│  Cons: Latency, single point of failure, scaling challenges             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    DISTRIBUTED AUTHORIZATION                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐      │
│  │    Service A     │  │    Service B     │  │    Service C     │      │
│  │  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │      │
│  │  │ Local PDP  │  │  │  │ Local PDP  │  │  │  │ Local PDP  │  │      │
│  │  │ (embedded) │  │  │  │ (embedded) │  │  │  │ (embedded) │  │      │
│  │  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │      │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘      │
│           │                    │                    │                    │
│           └────────────────────┼────────────────────┘                   │
│                                │                                         │
│                                ▼                                         │
│                    ┌────────────────────┐                               │
│                    │   Policy Store     │  ← Policies synced to all     │
│                    │   (Git, DB, etc.)  │  ← Services evaluate locally  │
│                    └────────────────────┘                               │
│                                                                          │
│  Pros: Low latency, no single point of failure, scales well            │
│  Cons: Policy sync complexity, eventual consistency                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5️⃣ Simulation-First Explanation

### Complete Authorization Flow

Let's trace what happens when Alice tries to delete an order:

**Step 1: Request Arrives**
```http
DELETE /api/orders/order-123 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

**Step 2: Authentication (Extract Identity)**
```java
// JWT contains:
{
  "sub": "user-alice",
  "email": "alice@example.com",
  "roles": ["manager"],
  "department": "sales"
}
```

**Step 3: PEP Intercepts and Builds Authorization Request**
```java
AuthorizationRequest request = AuthorizationRequest.builder()
    .subject(Subject.builder()
        .id("user-alice")
        .roles(List.of("manager"))
        .attributes(Map.of(
            "department", "sales",
            "clearance", 3
        ))
        .build())
    .action("delete")
    .resource(Resource.builder()
        .type("order")
        .id("order-123")
        .attributes(Map.of(
            "department", "sales",
            "status", "pending",
            "amount", 5000
        ))
        .build())
    .context(Context.builder()
        .timestamp(Instant.now())
        .ipAddress("10.0.0.1")
        .build())
    .build();
```

**Step 4: PDP Evaluates Policies**
```rego
# OPA Rego policy
package orders

default allow = false

# Managers can delete orders in their department
allow {
    input.action == "delete"
    input.resource.type == "order"
    input.subject.roles[_] == "manager"
    input.subject.department == input.resource.department
    input.resource.status == "pending"
}

# Admins can delete any order
allow {
    input.action == "delete"
    input.resource.type == "order"
    input.subject.roles[_] == "admin"
}

# Nobody can delete completed orders
deny {
    input.action == "delete"
    input.resource.type == "order"
    input.resource.status == "completed"
}
```

**Step 5: Decision Returned**
```json
{
  "decision": "ALLOW",
  "reason": "Manager can delete pending orders in their department",
  "policy": "orders/allow/rule2"
}
```

**Step 6: PEP Enforces Decision**
```java
if (decision.isAllowed()) {
    return orderService.deleteOrder("order-123");
} else {
    throw new ForbiddenException(decision.getReason());
}
```

---

## 6️⃣ How to Implement Authorization in Java

### Spring Security Method-Level Authorization

```java
package com.example.security.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler;
import org.springframework.security.access.expression.method.MethodSecurityExpressionHandler;

@Configuration
@EnableMethodSecurity(prePostEnabled = true)  // Enable @PreAuthorize, @PostAuthorize
public class MethodSecurityConfig {

    @Bean
    public MethodSecurityExpressionHandler methodSecurityExpressionHandler() {
        DefaultMethodSecurityExpressionHandler handler = 
            new DefaultMethodSecurityExpressionHandler();
        handler.setPermissionEvaluator(new CustomPermissionEvaluator());
        return handler;
    }
}
```

### Using @PreAuthorize Annotations

```java
package com.example.service;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.access.prepost.PostAuthorize;
import org.springframework.security.access.prepost.PostFilter;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    /**
     * Simple role-based check.
     * Only users with ADMIN role can call this method.
     */
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteAllOrders() {
        // Only admins can do this
    }

    /**
     * Multiple roles check.
     * ADMIN or MANAGER can access.
     */
    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
    public List<Order> getAllOrders() {
        return orderRepository.findAll();
    }

    /**
     * Permission-based check using SpEL.
     * User must have 'orders:read' permission.
     */
    @PreAuthorize("hasAuthority('orders:read')")
    public Order getOrder(String orderId) {
        return orderRepository.findById(orderId).orElseThrow();
    }

    /**
     * Resource-level authorization.
     * User can only access their own orders.
     * #userId refers to the method parameter.
     */
    @PreAuthorize("#userId == authentication.principal.id")
    public List<Order> getOrdersByUser(String userId) {
        return orderRepository.findByUserId(userId);
    }

    /**
     * Complex business rule.
     * Managers can update orders in their department.
     */
    @PreAuthorize("hasRole('MANAGER') and " +
                  "@orderAuthorizationService.canUpdateOrder(authentication, #orderId)")
    public Order updateOrder(String orderId, OrderUpdateRequest request) {
        return orderRepository.save(/* ... */);
    }

    /**
     * Post-authorization: Check after method executes.
     * Useful when you need the return value for the check.
     */
    @PostAuthorize("returnObject.userId == authentication.principal.id or hasRole('ADMIN')")
    public Order getOrderDetails(String orderId) {
        return orderRepository.findById(orderId).orElseThrow();
    }

    /**
     * Filter results based on authorization.
     * Returns only orders the user is allowed to see.
     */
    @PostFilter("filterObject.department == authentication.principal.department or hasRole('ADMIN')")
    public List<Order> searchOrders(OrderSearchCriteria criteria) {
        return orderRepository.search(criteria);
    }
}
```

### Custom Permission Evaluator

```java
package com.example.security;

import org.springframework.security.access.PermissionEvaluator;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Component;

import java.io.Serializable;

/**
 * Custom permission evaluator for fine-grained authorization.
 * 
 * Enables: @PreAuthorize("hasPermission(#order, 'delete')")
 */
@Component
public class CustomPermissionEvaluator implements PermissionEvaluator {

    private final OrderRepository orderRepository;

    public CustomPermissionEvaluator(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Override
    public boolean hasPermission(Authentication auth, Object targetObject, Object permission) {
        if (auth == null || targetObject == null || !(permission instanceof String)) {
            return false;
        }

        String permissionString = (String) permission;
        
        if (targetObject instanceof Order order) {
            return hasOrderPermission(auth, order, permissionString);
        }
        
        if (targetObject instanceof Document document) {
            return hasDocumentPermission(auth, document, permissionString);
        }
        
        return false;
    }

    @Override
    public boolean hasPermission(Authentication auth, Serializable targetId, 
                                  String targetType, Object permission) {
        if (auth == null || targetId == null || targetType == null) {
            return false;
        }

        // Load the object and check permission
        if ("Order".equals(targetType)) {
            Order order = orderRepository.findById((String) targetId).orElse(null);
            if (order == null) return false;
            return hasOrderPermission(auth, order, (String) permission);
        }
        
        return false;
    }

    private boolean hasOrderPermission(Authentication auth, Order order, String permission) {
        CustomUserDetails user = (CustomUserDetails) auth.getPrincipal();
        
        return switch (permission) {
            case "read" -> 
                // Owner can read, or admin, or same department manager
                order.getUserId().equals(user.getId()) ||
                user.hasRole("ADMIN") ||
                (user.hasRole("MANAGER") && order.getDepartment().equals(user.getDepartment()));
                
            case "update" -> 
                // Owner can update pending orders, or admin
                (order.getUserId().equals(user.getId()) && order.getStatus() == OrderStatus.PENDING) ||
                user.hasRole("ADMIN");
                
            case "delete" -> 
                // Only admin can delete, or owner of pending orders
                user.hasRole("ADMIN") ||
                (order.getUserId().equals(user.getId()) && order.getStatus() == OrderStatus.PENDING);
                
            default -> false;
        };
    }

    private boolean hasDocumentPermission(Authentication auth, Document doc, String permission) {
        // Similar logic for documents
        return false;
    }
}
```

### ABAC with Spring Security

```java
package com.example.security.abac;

import org.springframework.stereotype.Service;

import java.time.LocalTime;
import java.util.Map;

/**
 * ABAC Policy Decision Point.
 * Evaluates complex attribute-based policies.
 */
@Service
public class AbacPolicyEngine {

    /**
     * Evaluate an authorization request against ABAC policies.
     */
    public AuthorizationDecision evaluate(AuthorizationRequest request) {
        // Get all applicable policies
        List<Policy> policies = getPoliciesForResource(request.getResource().getType());
        
        for (Policy policy : policies) {
            PolicyResult result = evaluatePolicy(policy, request);
            
            if (result == PolicyResult.DENY) {
                return AuthorizationDecision.deny(policy.getName());
            }
            
            if (result == PolicyResult.ALLOW) {
                return AuthorizationDecision.allow(policy.getName());
            }
            // NOT_APPLICABLE continues to next policy
        }
        
        // Default deny if no policy matched
        return AuthorizationDecision.deny("No applicable policy");
    }

    private PolicyResult evaluatePolicy(Policy policy, AuthorizationRequest request) {
        // Evaluate all conditions in the policy
        for (Condition condition : policy.getConditions()) {
            if (!evaluateCondition(condition, request)) {
                return PolicyResult.NOT_APPLICABLE;
            }
        }
        
        return policy.getEffect();  // ALLOW or DENY
    }

    private boolean evaluateCondition(Condition condition, AuthorizationRequest request) {
        Object actualValue = getAttributeValue(condition.getAttribute(), request);
        Object expectedValue = condition.getValue();
        
        return switch (condition.getOperator()) {
            case EQUALS -> actualValue.equals(expectedValue);
            case NOT_EQUALS -> !actualValue.equals(expectedValue);
            case GREATER_THAN -> ((Comparable) actualValue).compareTo(expectedValue) > 0;
            case LESS_THAN -> ((Comparable) actualValue).compareTo(expectedValue) < 0;
            case IN -> ((Collection<?>) expectedValue).contains(actualValue);
            case CONTAINS -> ((Collection<?>) actualValue).contains(expectedValue);
            case BETWEEN -> {
                List<?> range = (List<?>) expectedValue;
                Comparable val = (Comparable) actualValue;
                yield val.compareTo(range.get(0)) >= 0 && val.compareTo(range.get(1)) <= 0;
            }
        };
    }

    private Object getAttributeValue(String attribute, AuthorizationRequest request) {
        String[] parts = attribute.split("\\.");
        String category = parts[0];
        String name = parts[1];
        
        return switch (category) {
            case "subject" -> request.getSubject().getAttribute(name);
            case "resource" -> request.getResource().getAttribute(name);
            case "action" -> request.getAction();
            case "environment" -> getEnvironmentAttribute(name);
            default -> throw new IllegalArgumentException("Unknown attribute category: " + category);
        };
    }

    private Object getEnvironmentAttribute(String name) {
        return switch (name) {
            case "currentTime" -> LocalTime.now();
            case "dayOfWeek" -> LocalDate.now().getDayOfWeek();
            case "ipAddress" -> RequestContextHolder.currentRequestAttributes()
                .getAttribute("clientIp", RequestAttributes.SCOPE_REQUEST);
            default -> null;
        };
    }
}
```

### Integration with OPA (Open Policy Agent)

```java
package com.example.security.opa;

import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

/**
 * Client for Open Policy Agent (OPA).
 * OPA is a general-purpose policy engine that uses Rego language.
 */
@Service
public class OpaClient {

    private final WebClient webClient;

    public OpaClient(@Value("${opa.url:http://localhost:8181}") String opaUrl) {
        this.webClient = WebClient.builder()
            .baseUrl(opaUrl)
            .build();
    }

    /**
     * Query OPA for an authorization decision.
     * 
     * @param policy The policy path (e.g., "orders/allow")
     * @param input The input data for policy evaluation
     * @return Authorization decision
     */
    public Mono<OpaDecision> query(String policy, OpaInput input) {
        return webClient.post()
            .uri("/v1/data/{policy}", policy)
            .bodyValue(Map.of("input", input))
            .retrieve()
            .bodyToMono(OpaResponse.class)
            .map(response -> new OpaDecision(
                response.getResult() != null && (Boolean) response.getResult(),
                policy
            ));
    }

    /**
     * Synchronous version for use in @PreAuthorize.
     */
    public boolean isAllowed(String policy, OpaInput input) {
        return query(policy, input)
            .map(OpaDecision::isAllowed)
            .block();
    }
}

/**
 * Input structure for OPA queries.
 */
@Data
@Builder
public class OpaInput {
    private Subject subject;
    private String action;
    private Resource resource;
    private Map<String, Object> context;
    
    @Data
    @Builder
    public static class Subject {
        private String id;
        private List<String> roles;
        private Map<String, Object> attributes;
    }
    
    @Data
    @Builder
    public static class Resource {
        private String type;
        private String id;
        private Map<String, Object> attributes;
    }
}
```

### OPA Rego Policies

```rego
# policies/orders.rego
package orders

import future.keywords.if
import future.keywords.in

default allow := false

# Rule 1: Admins can do anything
allow if {
    "admin" in input.subject.roles
}

# Rule 2: Users can read their own orders
allow if {
    input.action == "read"
    input.resource.type == "order"
    input.resource.attributes.userId == input.subject.id
}

# Rule 3: Managers can read orders in their department
allow if {
    input.action == "read"
    input.resource.type == "order"
    "manager" in input.subject.roles
    input.resource.attributes.department == input.subject.attributes.department
}

# Rule 4: Users can create orders
allow if {
    input.action == "create"
    input.resource.type == "order"
}

# Rule 5: Users can update their pending orders
allow if {
    input.action == "update"
    input.resource.type == "order"
    input.resource.attributes.userId == input.subject.id
    input.resource.attributes.status == "pending"
}

# Rule 6: Only during business hours (9am-6pm)
allow if {
    input.action in ["create", "update", "delete"]
    time.clock(time.now_ns())[0] >= 9
    time.clock(time.now_ns())[0] < 18
}

# Deny rule: Cannot delete completed orders
deny if {
    input.action == "delete"
    input.resource.attributes.status == "completed"
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Authorization Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Client-side only checks | Bypassed by API calls | Always check on server |
| Missing checks on some endpoints | Unauthorized access | Consistent enforcement |
| IDOR (Insecure Direct Object Reference) | Access others' data | Verify ownership |
| Role hierarchy bugs | Privilege escalation | Test role inheritance |
| Caching stale permissions | Outdated access | Short TTL, invalidation |
| Hardcoded roles in code | Can't change without deploy | Externalize policies |

### IDOR Prevention

```java
// WRONG: Trusts user-provided ID
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable String orderId) {
    return orderRepository.findById(orderId).orElseThrow();
}

// RIGHT: Verifies ownership
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable String orderId, Authentication auth) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    
    String userId = ((CustomUserDetails) auth.getPrincipal()).getId();
    if (!order.getUserId().equals(userId) && !hasRole(auth, "ADMIN")) {
        throw new ForbiddenException("Not authorized to access this order");
    }
    
    return order;
}

// BETTER: Use query that includes ownership check
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable String orderId, Authentication auth) {
    String userId = ((CustomUserDetails) auth.getPrincipal()).getId();
    return orderRepository.findByIdAndUserId(orderId, userId)
        .orElseThrow(() -> new NotFoundException("Order not found"));
}
```

---

## 8️⃣ When NOT to Use Complex Authorization

### When Simple RBAC is Enough

- Small applications with few roles
- Clear organizational hierarchy
- Static permission requirements
- No resource-level permissions needed

### When to Avoid Over-Engineering

- Don't use ABAC if RBAC suffices
- Don't use external policy engine for simple rules
- Don't implement ReBAC without a clear need for relationship-based access

---

## 9️⃣ Comparison of Authorization Models

| Aspect | RBAC | ABAC | ReBAC |
|--------|------|------|-------|
| Complexity | Low | High | Medium |
| Flexibility | Limited | Very High | High |
| Performance | Fast | Slower (evaluation) | Slower (graph traversal) |
| Audit | Easy | Complex | Medium |
| Use Case | Org hierarchy | Complex rules | Sharing/collaboration |
| Examples | Admin/User/Guest | Time-based, location-based | Google Docs, GitHub |

---

## 🔟 Interview Follow-Up Questions

### L4 (Entry Level) Questions

**Q: What's the difference between authentication and authorization?**
A: Authentication verifies WHO you are (identity), typically via username/password, tokens, or certificates. Authorization determines WHAT you can do (permissions) after your identity is verified. You must authenticate before authorization. Example: Authentication confirms you're "Alice". Authorization checks if Alice can delete documents.

**Q: What is RBAC and why is it commonly used?**
A: RBAC (Role-Based Access Control) assigns permissions to roles, and users are assigned roles. Instead of giving each user individual permissions, you assign them a role like "Admin" or "Editor" that comes with predefined permissions. It's common because it's simple to understand, easy to audit, maps well to organizational structures, and reduces permission management complexity.

### L5 (Mid Level) Questions

**Q: How would you prevent IDOR vulnerabilities?**
A: IDOR (Insecure Direct Object Reference) occurs when users access resources by manipulating IDs. Prevention:
1. Always verify ownership/authorization on the server, never trust client
2. Use queries that include ownership check: `findByIdAndUserId()`
3. Use non-guessable IDs (UUIDs instead of sequential)
4. Implement consistent authorization checks via interceptors/filters
5. Log and alert on authorization failures

**Q: Explain the difference between RBAC and ABAC with examples.**
A: RBAC makes decisions based on roles: "Managers can approve expenses." It's simple but can't express complex conditions.

ABAC makes decisions based on multiple attributes: "Approve if requester.department == expense.department AND expense.amount < requester.approvalLimit AND time is business hours." It's more flexible but complex.

Example scenario: "Doctors can view patient records" (RBAC) vs "Doctors can view patient records if they're the treating physician and the patient hasn't opted out and it's during the doctor's shift" (ABAC).

### L6 (Senior) Questions

**Q: Design an authorization system for a multi-tenant SaaS application.**
A:
1. **Tenant isolation**: Every resource belongs to a tenant, all queries filter by tenant
2. **Hierarchical RBAC**: Tenant admin > Department manager > User
3. **Cross-tenant access**: Super admin role for platform operators
4. **Resource-level permissions**: Share specific resources across tenants
5. **Policy engine**: OPA for complex policies, cached decisions
6. **Audit logging**: All authorization decisions logged
7. **API design**: Tenant ID in path or header, validated against token
8. **Performance**: Cache policies and decisions, invalidate on changes

**Q: How would you migrate from embedded authorization to a centralized policy engine?**
A:
1. **Inventory**: Document all existing authorization checks
2. **Standardize**: Create consistent authorization request format
3. **Parallel run**: New policy engine alongside existing checks
4. **Shadow mode**: Log decisions without enforcing
5. **Compare**: Verify policy engine matches existing behavior
6. **Gradual rollout**: Enable enforcement per service/endpoint
7. **Deprecate**: Remove old checks after validation
8. **Monitor**: Track decision latency, policy evaluation time

---

## 1️⃣1️⃣ One Clean Mental Summary

Authorization answers "What can this user do?" after authentication answers "Who is this user?" RBAC (Role-Based) assigns permissions to roles and roles to users, perfect for organizational hierarchies. ABAC (Attribute-Based) evaluates policies against user, resource, action, and environment attributes, enabling complex business rules. ReBAC (Relationship-Based) grants access based on relationships like ownership or sharing. For most applications, start with RBAC and add ABAC for specific complex rules. Always enforce authorization on the server, verify resource ownership to prevent IDOR, and consider centralizing policies for consistency across services.

---

## References

- [NIST RBAC Model](https://csrc.nist.gov/projects/role-based-access-control)
- [NIST ABAC Guide](https://nvlpubs.nist.gov/nistpubs/specialpublications/NIST.SP.800-162.pdf)
- [Open Policy Agent](https://www.openpolicyagent.org/)
- [Google Zanzibar Paper](https://research.google/pubs/pub48190/)
- [Spring Security Authorization](https://docs.spring.io/spring-security/reference/servlet/authorization/index.html)

