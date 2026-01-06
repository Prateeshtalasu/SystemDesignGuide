# 📐 API Design Principles: Building APIs That Last

## 0️⃣ Prerequisites

Before diving into API design, you should understand:

- **HTTP Protocol**: Methods (GET, POST, PUT, DELETE), status codes, headers. See `03-http-evolution.md`.

- **JSON**: JavaScript Object Notation, the standard data format for REST APIs.

- **Client-Server Architecture**: Clients make requests, servers respond with data.

- **URL Structure**: Protocol, domain, path, query parameters. Example: `https://api.example.com/users?limit=10`

---

## 1️⃣ What Problem Does Good API Design Solve?

### The Specific Pain Point

APIs are contracts between systems. A poorly designed API causes:
- Confusion for developers using it
- Breaking changes that disrupt clients
- Inconsistent behavior across endpoints
- Difficulty evolving the API over time
- Security vulnerabilities

**The Problem**: How do you design an API that's intuitive, consistent, evolvable, and secure?

### What Bad APIs Look Like

```mermaid
flowchart TD
    Bad["Bad API Design Examples"]
    
    Inconsistent["Inconsistent Naming:<br/>GET /getUsers<br/>GET /fetch_orders<br/>GET /ProductList<br/>GET /retrieve-customers"]
    
    ActionURL["Action in URL (not RESTful):<br/>POST /createUser<br/>POST /deleteUser/123<br/>GET /getUserById?id=123"]
    
    InconsistentResp["Inconsistent Response Structure:<br/>GET /users/123 → {'user': {'id': 123, 'name': 'Alice'}}<br/>GET /orders/456 → {'id': 456, 'total': 100} (no wrapper)<br/>GET /products/789 → [{'product_id': 789}] (array for single item?)"]
    
    UnclearError["Unclear Error Responses:<br/>{'error': true}<br/>{'message': 'Something went wrong'}<br/>{'code': -1}"]
    
    NoVersioning["No Versioning:<br/>Breaking changes affect all clients immediately"]
    
    Bad --> Inconsistent
    Bad --> ActionURL
    Bad --> InconsistentResp
    Bad --> UnclearError
    Bad --> NoVersioning
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    BAD API DESIGN EXAMPLES                                   │
└─────────────────────────────────────────────────────────────────────────────┘

Inconsistent Naming:
  GET /getUsers
  GET /fetch_orders
  GET /ProductList
  GET /retrieve-customers

Action in URL (not RESTful):
  POST /createUser
  POST /deleteUser/123
  GET /getUserById?id=123

Inconsistent Response Structure:
  GET /users/123    → {"user": {"id": 123, "name": "Alice"}}
  GET /orders/456   → {"id": 456, "total": 100}  (no wrapper)
  GET /products/789 → [{"product_id": 789}]      (array for single item?)

Unclear Error Responses:
  {"error": true}
  {"message": "Something went wrong"}
  {"code": -1}

No Versioning:
  Breaking changes affect all clients immediately
```
</details>

### Real Examples of the Problem

**Example 1: Twitter API v1**
Twitter's original API had inconsistent naming and response formats. They had to create v1.1 and v2 to fix design issues, breaking many applications.

**Example 2: Facebook Graph API**
Early versions had unclear rate limits and error codes. Developers couldn't reliably handle errors or plan for limits.

---

## 2️⃣ RESTful API Principles

### What is REST?

REST (Representational State Transfer) is an architectural style with these constraints:

1. **Client-Server**: Separation of concerns
2. **Stateless**: Each request contains all information needed
3. **Cacheable**: Responses can be cached
4. **Uniform Interface**: Consistent way to interact with resources
5. **Layered System**: Client doesn't know if it's talking to end server or intermediary

### Resource-Oriented Design

```mermaid
flowchart TD
    subgraph Resources["Resources are NOUNS, not verbs"]
        R1["✓ /users (collection of users)"]
        R2["✓ /users/123 (specific user)"]
        R3["✓ /users/123/orders (user's orders)"]
        R4["✗ /getUser"]
        R5["✗ /createUser"]
        R6["✗ /deleteUserById"]
    end
    
    subgraph Methods["HTTP Methods are VERBS"]
        M1["GET → Read (retrieve resource)"]
        M2["POST → Create (new resource)"]
        M3["PUT → Update (replace entire resource)"]
        M4["PATCH → Update (partial modification)"]
        M5["DELETE → Delete (remove resource)"]
    end
    
    subgraph Mapping["Mapping Actions to Methods"]
        Map1["Create user: POST /users"]
        Map2["List users: GET /users"]
        Map3["Get user: GET /users/123"]
        Map4["Update user: PUT /users/123"]
        Map5["Partial update: PATCH /users/123"]
        Map6["Delete user: DELETE /users/123"]
    end
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RESOURCE-ORIENTED DESIGN                                  │
└─────────────────────────────────────────────────────────────────────────────┘

Resources are NOUNS, not verbs:
─────────────────────────────────
✓ /users              (collection of users)
✓ /users/123          (specific user)
✓ /users/123/orders   (user's orders)
✗ /getUser
✗ /createUser
✗ /deleteUserById

HTTP Methods are VERBS:
─────────────────────────────────
GET     → Read (retrieve resource)
POST    → Create (new resource)
PUT     → Update (replace entire resource)
PATCH   → Update (partial modification)
DELETE  → Delete (remove resource)

Mapping Actions to Methods:
─────────────────────────────────
Create user:    POST   /users
List users:     GET    /users
Get user:       GET    /users/123
Update user:    PUT    /users/123
Partial update: PATCH  /users/123
Delete user:    DELETE /users/123
```
</details>

### URL Naming Conventions

```mermaid
flowchart TD
    BestPractices["URL Naming Best Practices"]
    
    P1["1. Use nouns, not verbs:<br/>✓ GET /articles<br/>✗ GET /getArticles"]
    P2["2. Use plural nouns for collections:<br/>✓ /users, /orders, /products<br/>✗ /user, /order, /product"]
    P3["3. Use lowercase with hyphens:<br/>✓ /user-profiles<br/>✗ /userProfiles, /user_profiles, /UserProfiles"]
    P4["4. Nest for relationships:<br/>✓ /users/123/orders (orders belonging to user 123)<br/>✓ /orders/456/items (items in order 456)"]
    P5["5. Limit nesting depth (max 2-3 levels):<br/>✓ /users/123/orders<br/>✗ /users/123/orders/456/items/789/reviews"]
    P6["6. Use query parameters for filtering:<br/>✓ /orders?status=pending&user_id=123<br/>✗ /orders/pending/user/123"]
    P7["7. Avoid file extensions:<br/>✓ /users/123 (use Accept header for format)<br/>✗ /users/123.json"]
    
    BestPractices --> P1 --> P2 --> P3 --> P4 --> P5 --> P6 --> P7
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    URL NAMING BEST PRACTICES                                 │
└─────────────────────────────────────────────────────────────────────────────┘

1. Use nouns, not verbs:
   ✓ GET /articles
   ✗ GET /getArticles

2. Use plural nouns for collections:
   ✓ /users, /orders, /products
   ✗ /user, /order, /product

3. Use lowercase with hyphens:
   ✓ /user-profiles
   ✗ /userProfiles, /user_profiles, /UserProfiles

4. Nest for relationships:
   ✓ /users/123/orders        (orders belonging to user 123)
   ✓ /orders/456/items        (items in order 456)
   
5. Limit nesting depth (max 2-3 levels):
   ✓ /users/123/orders
   ✗ /users/123/orders/456/items/789/reviews

6. Use query parameters for filtering:
   ✓ /orders?status=pending&user_id=123
   ✗ /orders/pending/user/123

7. Avoid file extensions:
   ✓ /users/123 (use Accept header for format)
   ✗ /users/123.json
```
</details>

### Complete CRUD Example

```mermaid
flowchart TD
    API["E-Commerce API Design"]
    
    subgraph Products["Products"]
        P1["GET /products<br/>List all products"]
        P2["GET /products?category=electronics&min_price=100"]
        P3["GET /products/123<br/>Get product 123"]
        P4["POST /products<br/>Create product"]
        P5["PUT /products/123<br/>Replace product 123"]
        P6["PATCH /products/123<br/>Update product 123 partially"]
        P7["DELETE /products/123<br/>Delete product 123"]
    end
    
    subgraph Orders["Orders"]
        O1["GET /orders<br/>List orders (for current user)"]
        O2["GET /orders?status=pending<br/>Filter by status"]
        O3["GET /orders/456<br/>Get order 456"]
        O4["POST /orders<br/>Create order"]
        O5["PATCH /orders/456<br/>Update order (e.g., cancel)"]
        O6["DELETE /orders/456<br/>Delete order (if allowed)"]
    end
    
    subgraph Items["Order Items (nested resource)"]
        I1["GET /orders/456/items<br/>List items in order 456"]
        I2["POST /orders/456/items<br/>Add item to order 456"]
        I3["DELETE /orders/456/items/789<br/>Remove item from order"]
    end
    
    subgraph UserOrders["User's Orders (relationship)"]
        U1["GET /users/123/orders<br/>List orders for user 123"]
    end
    
    API --> Products
    API --> Orders
    API --> Items
    API --> UserOrders
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    E-COMMERCE API DESIGN                                     │
└─────────────────────────────────────────────────────────────────────────────┘

Products:
─────────────────────────────────────────────────────────────────────────────
GET    /products                    List all products
GET    /products?category=electronics&min_price=100
GET    /products/123                Get product 123
POST   /products                    Create product
PUT    /products/123                Replace product 123
PATCH  /products/123                Update product 123 partially
DELETE /products/123                Delete product 123

Orders:
─────────────────────────────────────────────────────────────────────────────
GET    /orders                      List orders (for current user)
GET    /orders?status=pending       Filter by status
GET    /orders/456                  Get order 456
POST   /orders                      Create order
PATCH  /orders/456                  Update order (e.g., cancel)
DELETE /orders/456                  Delete order (if allowed)

Order Items (nested resource):
─────────────────────────────────────────────────────────────────────────────
GET    /orders/456/items            List items in order 456
POST   /orders/456/items            Add item to order 456
DELETE /orders/456/items/789        Remove item from order

User's Orders (relationship):
─────────────────────────────────────────────────────────────────────────────
GET    /users/123/orders            List orders for user 123
```
</details>

---

## 3️⃣ HTTP Status Codes

### Status Code Categories

```mermaid
flowchart TD
    Categories["HTTP Status Code Categories"]
    Info["1xx - Informational<br/>(rarely used in REST)"]
    Success["2xx - Success"]
    Redirect["3xx - Redirection"]
    ClientError["4xx - Client Error<br/>(client did something wrong)"]
    ServerError["5xx - Server Error<br/>(server did something wrong)"]
    
    Categories --> Info
    Categories --> Success
    Categories --> Redirect
    Categories --> ClientError
    Categories --> ServerError
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HTTP STATUS CODE CATEGORIES                               │
└─────────────────────────────────────────────────────────────────────────────┘

1xx - Informational (rarely used in REST)
2xx - Success
3xx - Redirection
4xx - Client Error (client did something wrong)
5xx - Server Error (server did something wrong)
```
</details>

### Complete Status Code Guide

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SUCCESS CODES (2xx)                                       │
└─────────────────────────────────────────────────────────────────────────────┘

200 OK
    - General success
    - GET: Resource retrieved
    - PUT/PATCH: Resource updated
    - Response includes body

201 Created
    - POST: Resource created successfully
    - Response should include created resource
    - Location header with URL to new resource

204 No Content
    - Success, but no body to return
    - DELETE: Resource deleted
    - PUT/PATCH when no response body needed

┌─────────────────────────────────────────────────────────────────────────────┐
│                    REDIRECTION CODES (3xx)                                   │
└─────────────────────────────────────────────────────────────────────────────┘

301 Moved Permanently
    - Resource URL has changed permanently
    - Include Location header with new URL

302 Found (Temporary Redirect)
    - Resource temporarily at different URL

304 Not Modified
    - Cached version is still valid
    - Used with If-None-Match / If-Modified-Since

┌─────────────────────────────────────────────────────────────────────────────┐
│                    CLIENT ERROR CODES (4xx)                                  │
└─────────────────────────────────────────────────────────────────────────────┘

400 Bad Request
    - Malformed request syntax
    - Invalid JSON
    - Missing required fields
    - Validation errors

401 Unauthorized
    - Authentication required
    - Invalid or missing credentials
    - Token expired

403 Forbidden
    - Authenticated but not authorized
    - User doesn't have permission

404 Not Found
    - Resource doesn't exist
    - Invalid URL

405 Method Not Allowed
    - HTTP method not supported for this resource
    - Include Allow header with valid methods

409 Conflict
    - Request conflicts with current state
    - Duplicate resource
    - Concurrent modification conflict

410 Gone
    - Resource permanently deleted
    - Different from 404 (existed but now gone)

422 Unprocessable Entity
    - Request syntax is correct
    - But semantic errors (business logic validation)
    - Example: "email already registered"

429 Too Many Requests
    - Rate limit exceeded
    - Include Retry-After header

┌─────────────────────────────────────────────────────────────────────────────┐
│                    SERVER ERROR CODES (5xx)                                  │
└─────────────────────────────────────────────────────────────────────────────┘

500 Internal Server Error
    - Generic server error
    - Unexpected exception

502 Bad Gateway
    - Upstream server returned invalid response
    - Proxy/gateway error

503 Service Unavailable
    - Server temporarily unavailable
    - Maintenance, overload
    - Include Retry-After header

504 Gateway Timeout
    - Upstream server didn't respond in time
```

### Status Code Decision Tree

```mermaid
flowchart TD
    Start["Request successful?"]
    Yes["Yes"]
    No["No"]
    
    Create["Creating resource?<br/>→ 201 Created"]
    Delete["Deleting resource?<br/>→ 204 No Content"]
    Other["Other<br/>→ 200 OK"]
    
    ClientFault["Client's fault?"]
    NotAuth["Not authenticated?<br/>→ 401 Unauthorized"]
    NotAuthorized["Not authorized?<br/>→ 403 Forbidden"]
    
    Start -->|Yes| Yes
    Start -->|No| No
    Yes --> Create
    Yes --> Delete
    Yes --> Other
    No --> ClientFault
    ClientFault --> NotAuth
    ClientFault --> NotAuthorized
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STATUS CODE DECISION TREE                                 │
└─────────────────────────────────────────────────────────────────────────────┘

Request successful?
├── Yes
│   ├── Creating resource? → 201 Created
│   ├── Deleting resource? → 204 No Content
│   └── Other → 200 OK
│
└── No
    ├── Client's fault?
    │   ├── Not authenticated? → 401 Unauthorized
    │   ├── Not authorized? → 403 Forbidden
```
</details>
    NotFound["Resource not found?<br/>→ 404 Not Found"]
    InvalidFormat["Invalid request format?<br/>→ 400 Bad Request"]
    ValidationError["Validation error?<br/>→ 422 Unprocessable Entity"]
    RateLimited["Rate limited?<br/>→ 429 Too Many Requests"]
    Conflict["Conflict?<br/>→ 409 Conflict"]
    
    ServerFault["Server's fault?"]
    Temporary["Temporary?<br/>→ 503 Service Unavailable"]
    Permanent["Permanent?<br/>→ 500 Internal Server Error"]
    
    ClientFault --> NotFound
    ClientFault --> InvalidFormat
    ClientFault --> ValidationError
    ClientFault --> RateLimited
    ClientFault --> Conflict
    
    No --> ServerFault
    ServerFault --> Temporary
    ServerFault --> Permanent
```
</details>

---

## 4️⃣ Request and Response Design

### Request Body Structure

```json
// POST /users - Create user
{
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "password": "securePassword123",
  "preferences": {
    "newsletter": true,
    "theme": "dark"
  }
}

// PATCH /users/123 - Partial update
{
  "email": "newemail@example.com"
}

// POST /orders - Create order
{
  "items": [
    {"product_id": "prod_123", "quantity": 2},
    {"product_id": "prod_456", "quantity": 1}
  ],
  "shipping_address": {
    "street": "123 Main St",
    "city": "San Francisco",
    "zip": "94102"
  }
}
```

### Response Structure

```json
// Successful single resource (GET /users/123)
{
  "data": {
    "id": "123",
    "type": "user",
    "attributes": {
      "name": "Alice Johnson",
      "email": "alice@example.com",
      "created_at": "2024-01-15T10:30:00Z"
    },
    "relationships": {
      "orders": {
        "links": {
          "related": "/users/123/orders"
        }
      }
    }
  },
  "links": {
    "self": "/users/123"
  }
}

// Simpler alternative (without JSON:API structure)
{
  "id": "123",
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "created_at": "2024-01-15T10:30:00Z"
}

// Collection response (GET /users)
{
  "data": [
    {"id": "123", "name": "Alice", "email": "alice@example.com"},
    {"id": "124", "name": "Bob", "email": "bob@example.com"}
  ],
  "meta": {
    "total": 100,
    "page": 1,
    "per_page": 20
  },
  "links": {
    "self": "/users?page=1",
    "next": "/users?page=2",
    "last": "/users?page=5"
  }
}
```

### Error Response Structure

```json
// Standard error format
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      },
      {
        "field": "password",
        "message": "Password must be at least 8 characters"
      }
    ],
    "request_id": "req_abc123",
    "documentation_url": "https://api.example.com/docs/errors#VALIDATION_ERROR"
  }
}

// Authentication error
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Invalid or expired token",
    "request_id": "req_def456"
  }
}

// Rate limit error
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests",
    "retry_after": 60,
    "request_id": "req_ghi789"
  }
}
```

---

## 5️⃣ API Versioning Strategies

### Why Version APIs?

APIs evolve. Breaking changes are sometimes necessary. Versioning lets you:
- Make breaking changes without breaking existing clients
- Support multiple versions during migration
- Deprecate old versions gracefully

### Versioning Strategies

```mermaid
flowchart TD
    Strategies["API Versioning Strategies"]
    
    URLPath["1. URL Path Versioning (Most Common)<br/>https://api.example.com/v1/users<br/>https://api.example.com/v2/users<br/>Pros: Clear, visible, easy to implement<br/>Cons: Changes URL, not truly RESTful<br/>Used by: Twitter, Stripe, GitHub"]
    
    QueryParam["2. Query Parameter Versioning<br/>https://api.example.com/users?version=1<br/>https://api.example.com/users?api-version=2024-01-01<br/>Pros: URL stays same<br/>Cons: Easy to forget, optional parameter issues<br/>Used by: Google, Amazon (some APIs)"]
    
    Header["3. Header Versioning<br/>GET /users<br/>Accept: application/vnd.example.v1+json<br/>Or custom header: X-API-Version: 1<br/>Pros: Clean URLs, truly RESTful<br/>Cons: Hidden, harder to test in browser<br/>Used by: GitHub (Accept header)"]
    
    ContentNeg["4. Content Negotiation<br/>Accept: application/vnd.example.user.v1+json<br/>Pros: Most RESTful<br/>Cons: Complex, not widely understood"]
    
    Strategies --> URLPath
    Strategies --> QueryParam
    Strategies --> Header
    Strategies --> ContentNeg
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    API VERSIONING STRATEGIES                                 │
└─────────────────────────────────────────────────────────────────────────────┘

1. URL Path Versioning (Most Common)
   ─────────────────────────────────────
   https://api.example.com/v1/users
   https://api.example.com/v2/users
   
   Pros: Clear, visible, easy to implement
   Cons: Changes URL, not truly RESTful
   Used by: Twitter, Stripe, GitHub

2. Query Parameter Versioning
   ─────────────────────────────────────
   https://api.example.com/users?version=1
   https://api.example.com/users?api-version=2024-01-01
   
   Pros: URL stays same
   Cons: Easy to forget, optional parameter issues
   Used by: Google, Amazon (some APIs)

3. Header Versioning
   ─────────────────────────────────────
   GET /users
   Accept: application/vnd.example.v1+json
   
   Or custom header:
   X-API-Version: 1
   
   Pros: Clean URLs, truly RESTful
   Cons: Hidden, harder to test in browser
   Used by: GitHub (Accept header)

4. Content Negotiation
   ─────────────────────────────────────
   Accept: application/vnd.example.user.v1+json
   
   Pros: Most RESTful
   Cons: Complex, not widely understood
```
</details>

### Version Lifecycle

```mermaid
gantt
    title API Version Lifecycle
    dateFormat YYYY-MM
    section v1
    Active Development    :2024-01, 5M
    Deprecated            :2024-06, 6M
    Sunset Warning        :2024-12, 3M
    Retired (410 Gone)    :2025-03, 0M
    section v2
    Current               :2024-06, 9M
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    API VERSION LIFECYCLE                                     │
└─────────────────────────────────────────────────────────────────────────────┘

Timeline:
─────────────────────────────────────────────────────────────────────────────

Jan 2024: v1 Released (Current)
          │
          │  [Active Development]
          │
Jun 2024: v2 Released (Current)
          v1 → Deprecated
          │
          │  [Migration Period - 6-12 months]
          │
Dec 2024: v1 → Sunset Warning
          │
          │  [Final Warning Period - 3 months]
          │
Mar 2025: v1 → Retired (returns 410 Gone)
          v2 → Current

Communication:
─────────────────────────────────────────────────────────────────────────────
1. Announce deprecation 6+ months in advance
2. Add Deprecation header to responses
3. Send emails to API users
4. Document migration path
5. Provide sunset date
6. Return 410 Gone after retirement
```
</details>

### Implementation Example

```java
// URL Path Versioning with Spring Boot
@RestController
@RequestMapping("/v1/users")
public class UserControllerV1 {
    
    @GetMapping("/{id}")
    public UserV1Response getUser(@PathVariable String id) {
        // V1 response format
        return new UserV1Response(userService.findById(id));
    }
}

@RestController
@RequestMapping("/v2/users")
public class UserControllerV2 {
    
    @GetMapping("/{id}")
    public UserV2Response getUser(@PathVariable String id) {
        // V2 response format (different structure)
        return new UserV2Response(userService.findById(id));
    }
}

// Header Versioning with Spring Boot
@RestController
@RequestMapping("/users")
public class UserController {
    
    @GetMapping(value = "/{id}", headers = "X-API-Version=1")
    public UserV1Response getUserV1(@PathVariable String id) {
        return new UserV1Response(userService.findById(id));
    }
    
    @GetMapping(value = "/{id}", headers = "X-API-Version=2")
    public UserV2Response getUserV2(@PathVariable String id) {
        return new UserV2Response(userService.findById(id));
    }
    
    // Default to latest version
    @GetMapping("/{id}")
    public UserV2Response getUser(@PathVariable String id) {
        return new UserV2Response(userService.findById(id));
    }
}
```

---

## 6️⃣ Pagination

### Why Pagination?

Returning thousands of records in one response:
- Slow response times
- High memory usage
- Poor user experience
- Wasted bandwidth

### Pagination Strategies

```mermaid
flowchart TD
    Offset["Offset-Based Pagination"]
    Request["Request:<br/>GET /users?limit=20&offset=40"]
    Response["Response:<br/>{<br/>  'data': [...],<br/>  'pagination': {<br/>    'total': 1000,<br/>    'limit': 20,<br/>    'offset': 40,<br/>    'has_more': true<br/>  }<br/>}"]
    Pros["Pros:<br/>- Simple to implement<br/>- Can jump to any page<br/>- Total count available"]
    Cons["Cons:<br/>- Slow for large offsets (OFFSET 10000 scans 10000 rows)<br/>- Inconsistent if data changes (items shift)<br/>- Can miss or duplicate items"]
    SQL["SQL: SELECT * FROM users LIMIT 20 OFFSET 40"]
    
    Offset --> Request --> Response
    Offset --> Pros
    Offset --> Cons
    Offset --> SQL
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OFFSET-BASED PAGINATION                                   │
└─────────────────────────────────────────────────────────────────────────────┘

Request:
  GET /users?limit=20&offset=40

Response:
{
  "data": [...],
  "pagination": {
    "total": 1000,
    "limit": 20,
    "offset": 40,
    "has_more": true
  }
}

Pros:
- Simple to implement
- Can jump to any page
- Total count available

Cons:
- Slow for large offsets (OFFSET 10000 scans 10000 rows)
- Inconsistent if data changes (items shift)
- Can miss or duplicate items

SQL: SELECT * FROM users LIMIT 20 OFFSET 40
```
</details>


```mermaid
flowchart TD
    Cursor["Cursor-Based Pagination"]
    Request2["Request:<br/>GET /users?limit=20&cursor=eyJpZCI6MTIzfQ==<br/><br/>Cursor is encoded: {'id': 123} → base64"]
    Response2["Response:<br/>{<br/>  'data': [...],<br/>  'pagination': {<br/>    'next_cursor': 'eyJpZCI6MTQzfQ==',<br/>    'has_more': true<br/>  }<br/>}"]
    Pros2["Pros:<br/>- Consistent results (no duplicates/misses)<br/>- Fast for any page (uses index)<br/>- Scales to millions of records"]
    Cons2["Cons:<br/>- Can't jump to arbitrary page<br/>- No total count (expensive to compute)<br/>- Cursor can become invalid"]
    SQL2["SQL: SELECT * FROM users WHERE id > 123 ORDER BY id LIMIT 20"]
    
    Cursor --> Request2 --> Response2
    Cursor --> Pros2
    Cursor --> Cons2
    Cursor --> SQL2
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CURSOR-BASED PAGINATION                                   │
└─────────────────────────────────────────────────────────────────────────────┘

Request:
  GET /users?limit=20&cursor=eyJpZCI6MTIzfQ==

Cursor is encoded: {"id": 123} → base64

Response:
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTQzfQ==",
    "has_more": true
  }
}

Pros:
- Consistent results (no duplicates/misses)
- Fast for any page (uses index)
- Scales to millions of records

Cons:
- Can't jump to arbitrary page
- No total count (expensive to compute)
- Cursor can become invalid

SQL: SELECT * FROM users WHERE id > 123 ORDER BY id LIMIT 20
```
</details>

### Cursor Implementation

```java
// Cursor-based pagination
@GetMapping("/users")
public PageResponse<UserDto> getUsers(
        @RequestParam(defaultValue = "20") int limit,
        @RequestParam(required = false) String cursor) {
    
    // Decode cursor
    Long afterId = null;
    if (cursor != null) {
        afterId = decodeCursor(cursor);
    }
    
    // Fetch limit + 1 to check if there's more
    List<User> users = userRepository.findUsersAfterId(afterId, limit + 1);
    
    boolean hasMore = users.size() > limit;
    if (hasMore) {
        users = users.subList(0, limit);
    }
    
    // Encode next cursor
    String nextCursor = null;
    if (hasMore && !users.isEmpty()) {
        User lastUser = users.get(users.size() - 1);
        nextCursor = encodeCursor(lastUser.getId());
    }
    
    return new PageResponse<>(
        users.stream().map(UserDto::from).toList(),
        new PaginationInfo(nextCursor, hasMore)
    );
}

private String encodeCursor(Long id) {
    return Base64.getEncoder().encodeToString(
        String.valueOf(id).getBytes()
    );
}

private Long decodeCursor(String cursor) {
    return Long.parseLong(
        new String(Base64.getDecoder().decode(cursor))
    );
}
```

### Pagination Comparison

| Feature | Offset-Based | Cursor-Based |
|---------|--------------|--------------|
| Jump to page | ✓ | ✗ |
| Total count | ✓ | ✗ (expensive) |
| Performance at scale | ✗ (slow) | ✓ (fast) |
| Consistent results | ✗ | ✓ |
| Implementation | Simple | More complex |
| Best for | Small datasets, admin UIs | Large datasets, infinite scroll |

---

## 7️⃣ Filtering, Sorting, and Field Selection

### Filtering

```mermaid
flowchart TD
    Filtering["Filtering Patterns"]
    
    Simple["Simple equality:<br/>GET /products?category=electronics&brand=apple"]
    
    Comparison["Comparison operators:<br/>GET /products?price[gte]=100&price[lte]=500<br/>GET /products?price_min=100&price_max=500<br/>GET /orders?created_after=2024-01-01"]
    
    Multiple["Multiple values (OR):<br/>GET /products?category=electronics,clothing<br/>GET /products?category[]=electronics&category[]=clothing"]
    
    Search["Search:<br/>GET /products?q=iphone<br/>GET /products?search=iphone+pro"]
    
    Complex["Complex filters (JSON):<br/>GET /products?filter={'category':'electronics','price':{'$gte':100}}"]
    
    Filtering --> Simple
    Filtering --> Comparison
    Filtering --> Multiple
    Filtering --> Search
    Filtering --> Complex
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FILTERING PATTERNS                                        │
└─────────────────────────────────────────────────────────────────────────────┘

Simple equality:
  GET /products?category=electronics&brand=apple

Comparison operators:
  GET /products?price[gte]=100&price[lte]=500
  GET /products?price_min=100&price_max=500
  GET /orders?created_after=2024-01-01

Multiple values (OR):
  GET /products?category=electronics,clothing
  GET /products?category[]=electronics&category[]=clothing

Search:
  GET /products?q=iphone
  GET /products?search=iphone+pro

Complex filters (JSON):
  GET /products?filter={"category":"electronics","price":{"$gte":100}}
```
</details>

### Sorting

```mermaid
flowchart TD
    Sorting["Sorting Patterns"]
    
    Single["Single field:<br/>GET /products?sort=price (ascending)<br/>GET /products?sort=-price (descending)<br/>GET /products?sort=price&order=desc"]
    
    Multiple2["Multiple fields:<br/>GET /products?sort=category,-price<br/>GET /products?sort[]=category&sort[]=-price"]
    
    Explicit["Explicit direction:<br/>GET /products?sort=price:asc,created_at:desc"]
    
    Sorting --> Single
    Sorting --> Multiple2
    Sorting --> Explicit
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SORTING PATTERNS                                          │
└─────────────────────────────────────────────────────────────────────────────┘

Single field:
  GET /products?sort=price           (ascending)
  GET /products?sort=-price          (descending)
  GET /products?sort=price&order=desc

Multiple fields:
  GET /products?sort=category,-price
  GET /products?sort[]=category&sort[]=-price

Explicit direction:
  GET /products?sort=price:asc,created_at:desc
```
</details>

### Field Selection (Sparse Fieldsets)

```mermaid
flowchart TD
    FieldSelection["Field Selection"]
    
    Select["Select specific fields:<br/>GET /users?fields=id,name,email<br/>GET /users?fields[user]=id,name&fields[posts]=title"]
    
    Without["Response without field selection:<br/>{<br/>  'id': '123',<br/>  'name': 'Alice',<br/>  'email': 'alice@example.com',<br/>  'address': {...},<br/>  'preferences': {...},<br/>  'created_at': '...'<br/>}"]
    
    With["Response with fields=id,name,email:<br/>{<br/>  'id': '123',<br/>  'name': 'Alice',<br/>  'email': 'alice@example.com'<br/>}"]
    
    FieldSelection --> Select
    Select --> Without
    Select --> With
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FIELD SELECTION                                           │
└─────────────────────────────────────────────────────────────────────────────┘

Select specific fields:
  GET /users?fields=id,name,email
  GET /users?fields[user]=id,name&fields[posts]=title

Response without field selection:
{
  "id": "123",
  "name": "Alice",
  "email": "alice@example.com",
  "address": {...},
  "preferences": {...},
  "created_at": "..."
}

Response with fields=id,name,email:
{
  "id": "123",
  "name": "Alice",
  "email": "alice@example.com"
}
```
</details>

### Implementation

```java
@GetMapping("/products")
public List<ProductDto> getProducts(
        @RequestParam(required = false) String category,
        @RequestParam(required = false) BigDecimal minPrice,
        @RequestParam(required = false) BigDecimal maxPrice,
        @RequestParam(required = false) String search,
        @RequestParam(defaultValue = "created_at") String sort,
        @RequestParam(defaultValue = "desc") String order,
        @RequestParam(required = false) Set<String> fields) {
    
    // Build specification for filtering
    Specification<Product> spec = Specification.where(null);
    
    if (category != null) {
        spec = spec.and((root, query, cb) -> 
            cb.equal(root.get("category"), category));
    }
    
    if (minPrice != null) {
        spec = spec.and((root, query, cb) -> 
            cb.greaterThanOrEqualTo(root.get("price"), minPrice));
    }
    
    if (maxPrice != null) {
        spec = spec.and((root, query, cb) -> 
            cb.lessThanOrEqualTo(root.get("price"), maxPrice));
    }
    
    if (search != null) {
        spec = spec.and((root, query, cb) -> 
            cb.like(cb.lower(root.get("name")), "%" + search.toLowerCase() + "%"));
    }
    
    // Build sort
    Sort sortObj = Sort.by(
        "desc".equalsIgnoreCase(order) ? Sort.Direction.DESC : Sort.Direction.ASC,
        sort
    );
    
    List<Product> products = productRepository.findAll(spec, sortObj);
    
    // Apply field selection
    return products.stream()
        .map(p -> ProductDto.from(p, fields))
        .toList();
}
```

---

## 8️⃣ HATEOAS (Hypermedia)

### What is HATEOAS?

HATEOAS (Hypermedia as the Engine of Application State) means responses include links to related actions and resources.

```json
// Without HATEOAS
{
  "id": "order_123",
  "status": "pending",
  "total": 99.99
}
// Client must know: How do I cancel? How do I pay?

// With HATEOAS
{
  "id": "order_123",
  "status": "pending",
  "total": 99.99,
  "links": {
    "self": {"href": "/orders/order_123"},
    "cancel": {"href": "/orders/order_123/cancel", "method": "POST"},
    "pay": {"href": "/orders/order_123/pay", "method": "POST"},
    "items": {"href": "/orders/order_123/items"}
  }
}
// Client discovers available actions from response
```

### HATEOAS Benefits

1. **Discoverability**: Client learns API by following links
2. **Decoupling**: URLs can change without breaking clients
3. **State-Driven**: Available actions depend on resource state

```json
// Order in "pending" state
{
  "id": "order_123",
  "status": "pending",
  "links": {
    "cancel": {"href": "/orders/order_123/cancel"},
    "pay": {"href": "/orders/order_123/pay"}
  }
}

// Order in "paid" state (different links!)
{
  "id": "order_123",
  "status": "paid",
  "links": {
    "track": {"href": "/orders/order_123/tracking"},
    "return": {"href": "/orders/order_123/return"}
  }
}
// No "cancel" or "pay" links because those actions aren't valid anymore
```

### Spring HATEOAS Implementation

```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    
    @GetMapping("/{id}")
    public EntityModel<OrderDto> getOrder(@PathVariable String id) {
        Order order = orderService.findById(id);
        OrderDto dto = OrderDto.from(order);
        
        EntityModel<OrderDto> model = EntityModel.of(dto);
        
        // Self link
        model.add(linkTo(methodOn(OrderController.class).getOrder(id)).withSelfRel());
        
        // Conditional links based on state
        if (order.getStatus() == OrderStatus.PENDING) {
            model.add(linkTo(methodOn(OrderController.class).cancelOrder(id)).withRel("cancel"));
            model.add(linkTo(methodOn(OrderController.class).payOrder(id)).withRel("pay"));
        }
        
        if (order.getStatus() == OrderStatus.PAID) {
            model.add(linkTo(methodOn(OrderController.class).trackOrder(id)).withRel("track"));
        }
        
        // Related resources
        model.add(linkTo(methodOn(OrderController.class).getOrderItems(id)).withRel("items"));
        
        return model;
    }
}
```

---

## 9️⃣ Security Best Practices

### Authentication

```mermaid
flowchart TD
    Auth["API Authentication Methods"]
    
    APIKey["1. API Keys (Simple, for server-to-server):<br/>X-API-Key: sk_live_abc123"]
    
    Bearer["2. Bearer Tokens (JWT, OAuth):<br/>Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."]
    
    Basic["3. Basic Auth (Legacy, over HTTPS only):<br/>Authorization: Basic base64(username:password)"]
    
    OAuth["4. OAuth 2.0 (Third-party access):<br/>Authorization: Bearer access_token"]
    
    Auth --> APIKey
    Auth --> Bearer
    Auth --> Basic
    Auth --> OAuth
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    API AUTHENTICATION METHODS                                │
└─────────────────────────────────────────────────────────────────────────────┘

1. API Keys (Simple, for server-to-server):
   X-API-Key: sk_live_abc123
   
2. Bearer Tokens (JWT, OAuth):
   Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
   
3. Basic Auth (Legacy, over HTTPS only):
   Authorization: Basic base64(username:password)
   
4. OAuth 2.0 (Third-party access):
   Authorization: Bearer access_token
```
</details>

### Security Headers

```java
@Configuration
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .headers(headers -> headers
                // Prevent clickjacking
                .frameOptions(frame -> frame.deny())
                // Prevent MIME sniffing
                .contentTypeOptions(content -> {})
                // XSS protection
                .xssProtection(xss -> xss.block(true))
                // HSTS
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000)
                    .includeSubDomains(true))
            )
            .csrf(csrf -> csrf.disable()) // Disable for stateless API
            .cors(cors -> cors.configurationSource(corsConfigurationSource()));
        
        return http.build();
    }
}
```

### Input Validation

```java
@PostMapping("/users")
public ResponseEntity<UserDto> createUser(
        @Valid @RequestBody CreateUserRequest request) {
    // @Valid triggers validation
    User user = userService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(UserDto.from(user));
}

public class CreateUserRequest {
    
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be 2-100 characters")
    private String name;
    
    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;
    
    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    @Pattern(regexp = ".*[A-Z].*", message = "Password must contain uppercase")
    @Pattern(regexp = ".*[0-9].*", message = "Password must contain number")
    private String password;
}
```

### Rate Limiting

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {
    
    private final RateLimiter rateLimiter; // e.g., Bucket4j, Resilience4j
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        
        String clientId = getClientId(request);
        
        if (!rateLimiter.tryConsume(clientId)) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setHeader("Retry-After", "60");
            response.getWriter().write("""
                {"error": {"code": "RATE_LIMIT_EXCEEDED", "message": "Too many requests"}}
                """);
            return;
        }
        
        // Add rate limit headers
        response.setHeader("X-RateLimit-Limit", "100");
        response.setHeader("X-RateLimit-Remaining", String.valueOf(rateLimiter.getRemaining(clientId)));
        response.setHeader("X-RateLimit-Reset", String.valueOf(rateLimiter.getResetTime(clientId)));
        
        filterChain.doFilter(request, response);
    }
}
```

---

## 🔟 Tradeoffs, Pitfalls, and Common Mistakes

### Pitfall 1: Inconsistent Naming

**Mistake**: Mixing naming conventions.

```
# BAD
GET /users
GET /getOrders
GET /product-list
GET /CustomerData

# GOOD
GET /users
GET /orders
GET /products
GET /customers
```

### Pitfall 2: Using Wrong HTTP Methods

**Mistake**: Using GET for operations with side effects.

```
# BAD
GET /users/123/delete
GET /orders/456/cancel

# GOOD
DELETE /users/123
POST /orders/456/cancel
```

### Pitfall 3: Not Using Proper Status Codes

**Mistake**: Always returning 200 with error in body.

```
# BAD
HTTP 200 OK
{"success": false, "error": "User not found"}

# GOOD
HTTP 404 Not Found
{"error": {"code": "NOT_FOUND", "message": "User not found"}}
```

### Pitfall 4: Breaking Changes Without Versioning

**Mistake**: Changing response structure without version bump.

```
# V1 Response
{"name": "Alice", "email": "alice@example.com"}

# Changed to (BREAKING!)
{"user": {"name": "Alice", "email": "alice@example.com"}}

# Clients expecting flat structure will break
```

### Pitfall 5: Exposing Internal IDs

**Mistake**: Using auto-increment database IDs in URLs.

```
# BAD - Reveals information, easy to enumerate
GET /users/1
GET /users/2
GET /users/3

# GOOD - Use UUIDs or opaque IDs
GET /users/usr_a1b2c3d4
GET /users/550e8400-e29b-41d4-a716-446655440000
```

---

## 1️⃣1️⃣ Interview Follow-Up Questions

### L4 (Junior/Mid) Level Questions

**Q1: What makes an API RESTful?**

**A**: REST APIs follow these principles:
1. **Resources**: URLs represent resources (nouns), not actions
2. **HTTP Methods**: Use GET, POST, PUT, PATCH, DELETE appropriately
3. **Stateless**: Each request contains all needed information
4. **Uniform Interface**: Consistent URL patterns and response formats

**Q2: When would you use PUT vs PATCH?**

**A**: 
- **PUT**: Replace entire resource. Client sends complete representation.
- **PATCH**: Partial update. Client sends only fields to change.

Example: Updating user email
- PUT: Send entire user object with new email
- PATCH: Send only `{"email": "new@example.com"}`

**Q3: How do you handle errors in a REST API?**

**A**: 
1. Use appropriate HTTP status codes (4xx for client errors, 5xx for server errors)
2. Return consistent error response structure
3. Include error code, message, and details
4. Add request ID for debugging
5. Don't expose internal errors to clients

### L5 (Senior) Level Questions

**Q4: How would you design versioning for a public API?**

**A**: 
1. **Strategy**: URL path versioning (`/v1/users`) for clarity
2. **Lifecycle**: 
   - Announce deprecation 6+ months ahead
   - Support 2 versions simultaneously
   - Add Deprecation header to old version
3. **Migration**: Provide migration guides, code examples
4. **Breaking Changes**: Only in new major versions
5. **Monitoring**: Track version usage to know when to sunset

**Q5: How do you design pagination for a large dataset?**

**A**: 
- **Cursor-based** for large datasets (millions of records)
- Encode cursor as opaque string (base64 of last ID)
- Use `WHERE id > cursor ORDER BY id LIMIT n`
- Return `next_cursor` and `has_more` in response
- Don't return total count (expensive query)

For admin UIs needing page numbers, use offset-based with caching.

### L6 (Staff+) Level Questions

**Q6: Design the API for a multi-tenant SaaS platform.**

**A**: 
1. **Tenant Isolation**: Tenant ID in URL (`/tenants/{id}/...`) or header
2. **Authentication**: JWT with tenant claim
3. **Rate Limiting**: Per-tenant limits
4. **Versioning**: URL path, support multiple versions
5. **Pagination**: Cursor-based for scalability
6. **Webhooks**: For async notifications
7. **Idempotency**: Idempotency keys for mutations
8. **Documentation**: OpenAPI spec, sandbox environment

**Q7: How do you evolve an API without breaking clients?**

**A**: 
1. **Additive Changes**: Add fields, don't remove
2. **Optional Fields**: New fields should be optional
3. **Deprecation**: Mark old fields deprecated before removal
4. **Versioning**: Major version for breaking changes
5. **Feature Flags**: Gradual rollout of new features
6. **Client Libraries**: Maintain official SDKs
7. **Communication**: Changelog, migration guides, email notifications

---

## 1️⃣2️⃣ Mental Summary

**Good API design** is about consistency, clarity, and evolvability. Use resources (nouns) in URLs, HTTP methods for actions, proper status codes for outcomes.

**Key principles**: RESTful design, consistent naming, proper status codes, versioning strategy, cursor-based pagination for scale, HATEOAS for discoverability.

**For production**: Validate all input, use proper authentication, implement rate limiting, return helpful error messages, document with OpenAPI.

**For interviews**: Know REST principles, HTTP methods and status codes, pagination strategies, versioning approaches, and how to handle breaking changes.

---

## 📚 Further Reading

- [REST API Design Best Practices](https://restfulapi.net/)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [Google API Design Guide](https://cloud.google.com/apis/design)
- [Stripe API Reference](https://stripe.com/docs/api) (excellent example)
- [JSON:API Specification](https://jsonapi.org/)
- [OpenAPI Specification](https://swagger.io/specification/)

