# ⚖️ Load Balancers: Distributing Traffic at Scale

## 0️⃣ Prerequisites

Before diving into load balancers, you should understand:

- **TCP/IP Basics**: How data travels over networks using IP addresses and ports. TCP provides reliable, ordered delivery. See `02-tcp-vs-udp.md` for details.

- **HTTP Protocol**: The request-response protocol used by web applications. Includes headers, methods (GET, POST), and status codes. See `03-http-evolution.md`.

- **DNS**: How domain names resolve to IP addresses. Load balancers often work in conjunction with DNS. See `01-dns-deep-dive.md`.

- **OSI Model**: The 7-layer networking model. Load balancers operate at Layer 4 (Transport) or Layer 7 (Application). Quick refresher:
  - Layer 4: TCP/UDP, sees IP addresses and ports
  - Layer 7: HTTP/HTTPS, sees URLs, headers, cookies

---

## 1️⃣ What Problem Do Load Balancers Exist to Solve?

### The Specific Pain Point

A single server has limits:
- **CPU**: Can only process so many requests per second
- **Memory**: Can only hold so much data
- **Network**: Can only handle so much bandwidth
- **Availability**: If it fails, everything is down

**The Problem**: How do you handle more traffic than one server can manage? How do you stay online when servers fail?

### What Systems Looked Like Before Load Balancers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SINGLE SERVER ARCHITECTURE                                │
└─────────────────────────────────────────────────────────────────────────────┘

                         ┌─────────────────┐
     All Users ─────────>│  Single Server  │
                         │                 │
                         │  - Web App      │
                         │  - Database     │
                         │  - Files        │
                         └─────────────────┘

Problems:
1. Server overloaded at peak traffic
2. Single point of failure (server down = site down)
3. Can't scale beyond server's capacity
4. Maintenance requires downtime
```

### What Breaks Without Load Balancers

**Without Load Balancing**:
- Popular sites crash during traffic spikes (Slashdot effect)
- Single server failure takes down entire service
- Can't deploy updates without downtime
- Geographic latency for distant users
- No way to scale horizontally

### Real Examples of the Problem

**Example 1: Twitter Fail Whale (2008-2012)**
Twitter's early architecture couldn't handle traffic spikes. Users saw the famous "Fail Whale" error page during major events. Load balancing and horizontal scaling eventually solved this.

**Example 2: Healthcare.gov Launch (2013)**
The site crashed on launch day due to overwhelming traffic. Lack of proper load balancing and capacity planning led to a $2 billion project failing publicly.

**Example 3: Amazon Prime Day**
Amazon handles 100x normal traffic during Prime Day. Without sophisticated load balancing, the site would crash within seconds.

---

## 2️⃣ Intuition and Mental Model

### The Restaurant Host Analogy

Think of a load balancer as a **restaurant host** managing multiple dining rooms.

**Without a host (no load balancer)**:
- All customers rush to the first dining room
- That room gets overcrowded while others sit empty
- If the first room closes, customers leave

**With a host (load balancer)**:
- Host greets customers at the entrance
- Directs them to available dining rooms based on capacity
- If one room closes for cleaning, redirects to others
- Ensures even distribution across all rooms

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LOAD BALANCER AS RESTAURANT HOST                          │
└─────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────┐
                              │    Host     │
                              │(Load Balancer)
                              └──────┬──────┘
                                     │
            ┌────────────────────────┼────────────────────────┐
            │                        │                        │
            ▼                        ▼                        ▼
    ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
    │  Dining Room  │       │  Dining Room  │       │  Dining Room  │
    │      A        │       │      B        │       │      C        │
    │  (Server 1)   │       │  (Server 2)   │       │  (Server 3)   │
    └───────────────┘       └───────────────┘       └───────────────┘
```

### The Key Insight

A load balancer is a **traffic cop** that:
1. Receives all incoming requests
2. Decides which backend server should handle each request
3. Forwards the request to the chosen server
4. Returns the response to the client

The client doesn't know (or care) which server handled their request.

---

## 3️⃣ How Load Balancers Work Internally

### Layer 4 (L4) vs Layer 7 (L7) Load Balancing

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    L4 vs L7 LOAD BALANCING                                   │
└─────────────────────────────────────────────────────────────────────────────┘

OSI Layer    What LB Sees           Decision Based On
─────────────────────────────────────────────────────────────────────────────
Layer 7      HTTP headers           URL path, cookies, headers, content type
(Application) Request body          User session, API version, authentication
             URL path               
             Cookies                Example: Route /api/* to API servers
                                            Route /static/* to CDN

Layer 4      TCP/UDP packets        Source IP, destination IP
(Transport)  IP addresses           Source port, destination port
             Port numbers           
                                    Example: Route port 443 to HTTPS servers
                                            Route port 3306 to DB servers
```

**L4 Load Balancer**:
- Operates at TCP/UDP level
- Faster (less processing)
- Can't make decisions based on content
- Good for: non-HTTP traffic, raw throughput

**L7 Load Balancer**:
- Operates at HTTP level
- Can inspect headers, URLs, cookies
- More flexible routing decisions
- Good for: web applications, API routing, A/B testing

### Load Balancing Algorithms

#### 1. Round Robin

Simplest algorithm. Distribute requests in circular order.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  (back to start)
Request 5 → Server B
...
```

**Pros**: Simple, even distribution
**Cons**: Ignores server capacity and current load

```java
public class RoundRobinBalancer {
    private final List<Server> servers;
    private final AtomicInteger counter = new AtomicInteger(0);
    
    public Server getNextServer() {
        int index = counter.getAndIncrement() % servers.size();
        return servers.get(index);
    }
}
```

#### 2. Weighted Round Robin

Like round robin, but servers with higher weights get more requests.

```
Servers: A (weight=3), B (weight=2), C (weight=1)

Request sequence: A, A, A, B, B, C, A, A, A, B, B, C, ...

Server A gets 3x the traffic of Server C
```

**Use case**: When servers have different capacities (e.g., 8-core vs 4-core machines).

```java
public class WeightedRoundRobinBalancer {
    private final List<WeightedServer> servers;
    private int currentIndex = 0;
    private int currentWeight = 0;
    
    public Server getNextServer() {
        while (true) {
            currentIndex = (currentIndex + 1) % servers.size();
            if (currentIndex == 0) {
                currentWeight--;
                if (currentWeight <= 0) {
                    currentWeight = getMaxWeight();
                }
            }
            
            WeightedServer server = servers.get(currentIndex);
            if (server.getWeight() >= currentWeight) {
                return server;
            }
        }
    }
}
```

#### 3. Least Connections

Route to the server with fewest active connections.

```
Server A: 10 active connections
Server B: 5 active connections   ← Next request goes here
Server C: 8 active connections
```

**Pros**: Adapts to server load in real-time
**Cons**: Doesn't account for request complexity

```java
public class LeastConnectionsBalancer {
    private final List<Server> servers;
    
    public synchronized Server getNextServer() {
        return servers.stream()
            .min(Comparator.comparingInt(Server::getActiveConnections))
            .orElseThrow();
    }
}
```

#### 4. Weighted Least Connections

Combines weights with connection count.

```
Score = Active Connections / Weight

Server A: 10 connections, weight 5 → Score = 2.0
Server B: 6 connections, weight 2  → Score = 3.0
Server C: 4 connections, weight 1  → Score = 4.0

Server A has lowest score, gets next request
```

#### 5. IP Hash (Source IP Affinity)

Hash the client IP to determine server. Same client always goes to same server.

```
hash("192.168.1.100") % 3 = 1 → Server B
hash("192.168.1.101") % 3 = 0 → Server A
hash("192.168.1.102") % 3 = 2 → Server C

Client 192.168.1.100 will ALWAYS go to Server B
```

**Pros**: Session persistence without cookies
**Cons**: Uneven distribution if many clients behind same NAT

```java
public class IPHashBalancer {
    private final List<Server> servers;
    
    public Server getServer(String clientIP) {
        int hash = clientIP.hashCode();
        int index = Math.abs(hash) % servers.size();
        return servers.get(index);
    }
}
```

#### 6. Consistent Hashing

Advanced hashing that minimizes redistribution when servers change.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONSISTENT HASHING RING                                   │
└─────────────────────────────────────────────────────────────────────────────┘

                           0°
                           │
                    Server A (30°)
                         ╱
                        ╱
               270° ───●─────────── 90°
                        ╲
                         ╲
                    Server C (200°)
                           │
                    Server B (150°)
                          180°

Request hash = 100° → Goes to Server B (next server clockwise)
Request hash = 220° → Goes to Server A (next server clockwise)

If Server B is removed:
- Only requests between 90° and 150° are redistributed
- Other requests unaffected
```

**Pros**: Minimal redistribution when scaling
**Cons**: More complex to implement

#### 7. Least Response Time

Route to server with fastest recent response times.

```
Server A: avg response 50ms
Server B: avg response 30ms  ← Next request goes here
Server C: avg response 45ms
```

**Pros**: Optimizes for user experience
**Cons**: Requires response time tracking

#### 8. Random

Randomly select a server. Surprisingly effective at scale.

```java
public class RandomBalancer {
    private final List<Server> servers;
    private final Random random = new Random();
    
    public Server getNextServer() {
        return servers.get(random.nextInt(servers.size()));
    }
}
```

**Pros**: Simple, stateless, no coordination needed
**Cons**: Can be uneven with small request counts

### Algorithm Comparison

| Algorithm | Best For | Avoids |
|-----------|----------|--------|
| Round Robin | Homogeneous servers, stateless apps | When servers have different capacities |
| Weighted Round Robin | Heterogeneous servers | When load varies significantly |
| Least Connections | Long-lived connections | When connections are very short |
| IP Hash | Session affinity needed | When clients are behind NAT |
| Consistent Hashing | Caching layers, stateful services | Simple stateless applications |
| Least Response Time | Latency-sensitive applications | When response times are uniform |

---

## 4️⃣ Health Checks

Load balancers must know which servers are healthy. Two approaches:

### Active Health Checks

Load balancer periodically probes each server.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ACTIVE HEALTH CHECK FLOW                                  │
└─────────────────────────────────────────────────────────────────────────────┘

Load Balancer                                          Backend Server
      │                                                      │
      │  GET /health (every 5 seconds)                       │
      │ ────────────────────────────────────────────────────>│
      │                                                      │
      │  200 OK {"status": "healthy", "db": "ok"}            │
      │ <────────────────────────────────────────────────────│
      │                                                      │
      │  [Server marked HEALTHY]                             │
      │                                                      │
      │  ... 5 seconds later ...                             │
      │                                                      │
      │  GET /health                                         │
      │ ────────────────────────────────────────────────────>│
      │                                                      │
      │  [Timeout - no response]                             │
      │                                                      │
      │  [Failure count: 1/3]                                │
      │                                                      │
      │  ... 5 seconds later ...                             │
      │                                                      │
      │  GET /health                                         │
      │ ────────────────────────────────────────────────────>│
      │                                                      │
      │  503 Service Unavailable                             │
      │ <────────────────────────────────────────────────────│
      │                                                      │
      │  [Failure count: 2/3]                                │
      │                                                      │
      │  ... after 3 failures ...                            │
      │                                                      │
      │  [Server marked UNHEALTHY - removed from pool]       │
```

**Configuration Parameters**:
- **Interval**: How often to check (e.g., 5 seconds)
- **Timeout**: How long to wait for response (e.g., 2 seconds)
- **Unhealthy threshold**: Failures before marking unhealthy (e.g., 3)
- **Healthy threshold**: Successes before marking healthy again (e.g., 2)

### Passive Health Checks

Monitor real traffic for failures. No extra probe requests.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PASSIVE HEALTH CHECK FLOW                                 │
└─────────────────────────────────────────────────────────────────────────────┘

Client Request → Load Balancer → Server A
                      │
                      │  [Server A returns 500 error]
                      │  [Error count for A: 1]
                      │
Client Request → Load Balancer → Server A
                      │
                      │  [Server A returns 502 error]
                      │  [Error count for A: 2]
                      │
                      │  ... after N errors in time window ...
                      │
                      │  [Server A marked UNHEALTHY]
                      │
Client Request → Load Balancer → Server B (A skipped)
```

**Pros**: No extra traffic, detects real failures
**Cons**: Clients experience failures before detection

### Health Check Endpoint Design

```java
@RestController
public class HealthController {
    
    @Autowired
    private DataSource dataSource;
    
    @Autowired
    private RedisTemplate<String, String> redis;
    
    /**
     * Simple liveness check - is the process running?
     */
    @GetMapping("/health/live")
    public ResponseEntity<String> liveness() {
        return ResponseEntity.ok("OK");
    }
    
    /**
     * Readiness check - can this server handle requests?
     */
    @GetMapping("/health/ready")
    public ResponseEntity<Map<String, Object>> readiness() {
        Map<String, Object> health = new HashMap<>();
        boolean healthy = true;
        
        // Check database
        try {
            dataSource.getConnection().isValid(2);
            health.put("database", "OK");
        } catch (Exception e) {
            health.put("database", "FAILED: " + e.getMessage());
            healthy = false;
        }
        
        // Check Redis
        try {
            redis.getConnectionFactory().getConnection().ping();
            health.put("redis", "OK");
        } catch (Exception e) {
            health.put("redis", "FAILED: " + e.getMessage());
            healthy = false;
        }
        
        // Check disk space
        File root = new File("/");
        long freeSpace = root.getFreeSpace();
        long totalSpace = root.getTotalSpace();
        double freePercent = (double) freeSpace / totalSpace * 100;
        
        if (freePercent < 10) {
            health.put("disk", "WARNING: " + freePercent + "% free");
            healthy = false;
        } else {
            health.put("disk", "OK: " + freePercent + "% free");
        }
        
        health.put("status", healthy ? "HEALTHY" : "UNHEALTHY");
        
        return healthy 
            ? ResponseEntity.ok(health)
            : ResponseEntity.status(503).body(health);
    }
}
```

---

## 5️⃣ Session Persistence (Sticky Sessions)

Some applications need requests from the same user to go to the same server.

### Why Session Persistence?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PROBLEM: STATEFUL SESSIONS                                │
└─────────────────────────────────────────────────────────────────────────────┘

Without Sticky Sessions:
─────────────────────────
Request 1 (Login) → Server A → Creates session in memory
Request 2 (Dashboard) → Server B → No session! User logged out!

With Sticky Sessions:
─────────────────────────
Request 1 (Login) → Server A → Creates session in memory
Request 2 (Dashboard) → Server A → Session found! User stays logged in
Request 3 (Profile) → Server A → Same server, session intact
```

### Methods of Session Persistence

#### 1. Cookie-Based (Application Cookie)

Load balancer inserts a cookie identifying the server.

```
Response from Server A:
Set-Cookie: SERVERID=server-a; Path=/

Subsequent requests include:
Cookie: SERVERID=server-a

Load balancer reads cookie, routes to Server A
```

#### 2. Source IP Affinity

Use client IP to determine server (IP Hash algorithm).

```
Client IP: 192.168.1.100 → Always routes to Server B
```

**Problem**: Clients behind NAT share IP, all go to same server.

#### 3. Application-Level Session ID

Application generates session ID, load balancer uses it for routing.

```
Application sets: Set-Cookie: JSESSIONID=abc123

Load balancer maintains mapping:
abc123 → Server A
def456 → Server B
```

### Better Alternative: Externalized Sessions

Instead of sticky sessions, store sessions externally.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EXTERNALIZED SESSION STORAGE                              │
└─────────────────────────────────────────────────────────────────────────────┘

                         ┌─────────────────┐
                         │  Load Balancer  │
                         └────────┬────────┘
                                  │
            ┌─────────────────────┼─────────────────────┐
            │                     │                     │
            ▼                     ▼                     ▼
    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
    │   Server A    │    │   Server B    │    │   Server C    │
    └───────┬───────┘    └───────┬───────┘    └───────┬───────┘
            │                     │                     │
            └─────────────────────┼─────────────────────┘
                                  │
                         ┌────────▼────────┐
                         │  Redis Cluster  │
                         │  (Session Store)│
                         └─────────────────┘

Any server can handle any request - sessions stored in Redis
```

```java
// Spring Session with Redis
@Configuration
@EnableRedisHttpSession
public class SessionConfig {
    
    @Bean
    public LettuceConnectionFactory connectionFactory() {
        return new LettuceConnectionFactory("redis-host", 6379);
    }
}

// application.yml
spring:
  session:
    store-type: redis
    redis:
      flush-mode: on_save
      namespace: spring:session
```

---

## 6️⃣ Connection Draining (Graceful Shutdown)

When removing a server, existing connections should complete gracefully.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONNECTION DRAINING                                       │
└─────────────────────────────────────────────────────────────────────────────┘

Time 0: Server A marked for removal
        ┌─────────────────────────────────────────────────────────────────┐
        │ Server A: 100 active connections                                │
        │ Status: DRAINING (no new connections, existing continue)        │
        └─────────────────────────────────────────────────────────────────┘

Time 30s: Connections completing
        ┌─────────────────────────────────────────────────────────────────┐
        │ Server A: 20 active connections                                 │
        │ Status: DRAINING                                                │
        └─────────────────────────────────────────────────────────────────┘

Time 60s: All connections complete or timeout
        ┌─────────────────────────────────────────────────────────────────┐
        │ Server A: 0 active connections                                  │
        │ Status: REMOVED (safe to shut down)                             │
        └─────────────────────────────────────────────────────────────────┘
```

**Configuration**:
- **Drain timeout**: Maximum time to wait (e.g., 60 seconds)
- **New connection behavior**: Reject or redirect to other servers

---

## 7️⃣ Simulation: Load Balancing in Action

Let's trace 10 requests through different algorithms.

### Scenario Setup

```
Servers:
- Server A: 4 CPU cores, weight=2
- Server B: 2 CPU cores, weight=1
- Server C: 2 CPU cores, weight=1
```

### Round Robin

```
Request 1  → Server A
Request 2  → Server B
Request 3  → Server C
Request 4  → Server A
Request 5  → Server B
Request 6  → Server C
Request 7  → Server A
Request 8  → Server B
Request 9  → Server C
Request 10 → Server A

Distribution: A=4, B=3, C=3 (equal, ignores weights)
```

### Weighted Round Robin

```
Request 1  → Server A (weight 2)
Request 2  → Server A (weight 2)
Request 3  → Server B (weight 1)
Request 4  → Server C (weight 1)
Request 5  → Server A
Request 6  → Server A
Request 7  → Server B
Request 8  → Server C
Request 9  → Server A
Request 10 → Server A

Distribution: A=6, B=2, C=2 (matches weights 2:1:1)
```

### Least Connections (Dynamic)

```
Initial: A=0, B=0, C=0

Request 1  → Server A (all equal, pick first)  → A=1, B=0, C=0
Request 2  → Server B (B and C tied, pick B)   → A=1, B=1, C=0
Request 3  → Server C (C lowest)               → A=1, B=1, C=1
Request 4  → Server A (all equal)              → A=2, B=1, C=1
            [Request 1 completes]              → A=1, B=1, C=1
Request 5  → Server A (all equal)              → A=2, B=1, C=1
Request 6  → Server B (B and C tied)           → A=2, B=2, C=1
Request 7  → Server C (C lowest)               → A=2, B=2, C=2
            [Request 3 completes]              → A=2, B=2, C=1
Request 8  → Server C (C lowest)               → A=2, B=2, C=2
...

Distribution: Varies based on request duration
```

---

## 8️⃣ How Engineers Use Load Balancers in Production

### Real-World Usage

**Netflix**
- Uses AWS ELB (Elastic Load Balancer) for external traffic
- Custom internal load balancer (Eureka + Ribbon) for service-to-service
- Handles millions of requests per second

**Google**
- Custom load balancers (Maglev) for global traffic
- Uses consistent hashing for cache efficiency
- Reference: [Maglev Paper](https://research.google/pubs/pub44824/)

**Uber**
- HAProxy for edge load balancing
- Custom load balancing for internal services
- Handles millions of trips per day

### Common Load Balancer Products

| Product | Type | Use Case |
|---------|------|----------|
| **AWS ALB** | L7, managed | Web applications, microservices |
| **AWS NLB** | L4, managed | High throughput, TCP/UDP |
| **AWS ELB Classic** | L4/L7, managed | Legacy applications |
| **Nginx** | L7, software | Reverse proxy, web serving |
| **HAProxy** | L4/L7, software | High performance, flexibility |
| **Envoy** | L7, software | Service mesh, modern apps |
| **F5 BIG-IP** | L4/L7, hardware | Enterprise, high security |
| **Cloudflare** | L7, CDN | DDoS protection, global |

### Production Configuration Examples

**Nginx Load Balancer**:

```nginx
# /etc/nginx/nginx.conf

upstream backend {
    # Load balancing algorithm
    least_conn;  # or: round_robin (default), ip_hash, hash
    
    # Backend servers with weights
    server backend1.example.com:8080 weight=3;
    server backend2.example.com:8080 weight=2;
    server backend3.example.com:8080 weight=1;
    
    # Backup server (only used when others are down)
    server backend4.example.com:8080 backup;
    
    # Health check parameters
    server backend5.example.com:8080 max_fails=3 fail_timeout=30s;
    
    # Keep connections alive to backends
    keepalive 32;
}

server {
    listen 80;
    listen 443 ssl;
    server_name example.com;
    
    ssl_certificate /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;
    
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffering
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
    
    # Health check endpoint (Nginx Plus feature)
    location /health {
        return 200 "OK";
    }
}
```

**HAProxy Configuration**:

```haproxy
# /etc/haproxy/haproxy.cfg

global
    log stdout format raw local0
    maxconn 4096
    
defaults
    mode http
    log global
    option httplog
    option dontlognull
    timeout connect 5s
    timeout client 50s
    timeout server 50s
    
frontend http_front
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/example.pem
    
    # ACL for routing
    acl is_api path_beg /api
    acl is_static path_beg /static
    
    # Route based on path
    use_backend api_servers if is_api
    use_backend static_servers if is_static
    default_backend web_servers
    
backend web_servers
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200
    
    server web1 192.168.1.10:8080 check weight 3
    server web2 192.168.1.11:8080 check weight 2
    server web3 192.168.1.12:8080 check weight 1 backup
    
backend api_servers
    balance leastconn
    option httpchk GET /api/health
    
    server api1 192.168.1.20:8080 check
    server api2 192.168.1.21:8080 check
    
backend static_servers
    balance roundrobin
    server static1 192.168.1.30:80 check
    server static2 192.168.1.31:80 check

# Stats page
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
```

**AWS Application Load Balancer (Terraform)**:

```hcl
# ALB
resource "aws_lb" "main" {
  name               = "main-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
  
  enable_deletion_protection = true
  
  access_logs {
    bucket  = aws_s3_bucket.lb_logs.bucket
    prefix  = "alb"
    enabled = true
  }
}

# Target Group
resource "aws_lb_target_group" "app" {
  name     = "app-tg"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  
  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200"
  }
  
  stickiness {
    type            = "lb_cookie"
    cookie_duration = 86400
    enabled         = true
  }
}

# Listener
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = aws_acm_certificate.main.arn
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# Path-based routing
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100
  
  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
  
  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }
}
```

---

## 9️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Pitfall 1: Single Load Balancer as Single Point of Failure

**Scenario**: Load balancer goes down, entire service is unavailable.

**Mistake**: Running only one load balancer instance.

**Solution**: High availability setup with multiple load balancers.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HIGH AVAILABILITY LOAD BALANCER                           │
└─────────────────────────────────────────────────────────────────────────────┘

                         ┌─────────────────┐
                         │   DNS (Route53) │
                         │ Multiple A records
                         └────────┬────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
           ┌───────────────┐           ┌───────────────┐
           │  LB Primary   │◄─────────►│  LB Secondary │
           │   (Active)    │  Heartbeat│   (Standby)   │
           └───────┬───────┘           └───────────────┘
                   │                           │
                   │  Virtual IP (VIP)         │
                   │  Failover via VRRP/Keepalived
                   │                           │
            ┌──────┴──────┐                    │
            │             │                    │
            ▼             ▼                    │
       ┌─────────┐   ┌─────────┐              │
       │Server A │   │Server B │◄─────────────┘
       └─────────┘   └─────────┘
```

### Pitfall 2: Not Preserving Client IP

**Scenario**: Backend servers see load balancer IP instead of client IP.

**Mistake**: Not forwarding original client IP.

**Solution**: Use X-Forwarded-For header.

```nginx
# Nginx configuration
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

```java
// Backend application
@GetMapping("/api/data")
public ResponseEntity<?> getData(HttpServletRequest request) {
    // Get real client IP
    String clientIP = request.getHeader("X-Forwarded-For");
    if (clientIP == null) {
        clientIP = request.getRemoteAddr();
    }
    // Use first IP if multiple (proxy chain)
    if (clientIP.contains(",")) {
        clientIP = clientIP.split(",")[0].trim();
    }
    
    log.info("Request from: {}", clientIP);
    // ...
}
```

### Pitfall 3: Improper Health Check Configuration

**Scenario**: Healthy servers marked unhealthy, or unhealthy servers kept in rotation.

**Mistakes**:
- Health check timeout too short (network latency causes false failures)
- Health check interval too long (slow to detect failures)
- Health check endpoint too simple (doesn't check dependencies)

**Solution**: Proper health check tuning.

```yaml
# Good health check configuration
health_check:
  path: /health/ready  # Checks dependencies, not just /health
  interval: 10s        # Check every 10 seconds
  timeout: 5s          # Wait up to 5 seconds
  healthy_threshold: 2     # 2 successes to mark healthy
  unhealthy_threshold: 3   # 3 failures to mark unhealthy
```

### Pitfall 4: Sticky Sessions Breaking Failover

**Scenario**: Server fails, but clients with sticky sessions can't be redistributed.

**Mistake**: Relying on server-local sessions with sticky sessions.

**Solution**: Externalize sessions (Redis) or make application stateless.

### Pitfall 5: Not Accounting for Slow Starts

**Scenario**: New server added, immediately overwhelmed with traffic.

**Mistake**: Adding server at full weight immediately.

**Solution**: Slow start / warm-up period.

```nginx
# Nginx slow start
upstream backend {
    server backend1.example.com:8080 weight=3;
    server backend2.example.com:8080 weight=3 slow_start=30s;  # Ramp up over 30s
}
```

---

## 🔟 When NOT to Use Load Balancers

### Scenarios Where Load Balancers May Be Overkill

| Scenario | Why | Alternative |
|----------|-----|-------------|
| Single server application | No servers to balance | Direct access |
| Development environment | Complexity not needed | Docker Compose |
| Batch processing jobs | Not request-based | Job scheduler |
| Internal tools with few users | Low traffic | Single server |

### When DNS Load Balancing Suffices

- Simple round-robin distribution
- Geographic routing (GeoDNS)
- No need for health checks
- Static server pool

### When Service Mesh Replaces Load Balancer

For microservices, service mesh (Istio, Linkerd) provides:
- Client-side load balancing
- Built-in retries and circuit breakers
- Observability
- mTLS

---

## 1️⃣1️⃣ Comparison: Load Balancer Types

| Feature | L4 (NLB) | L7 (ALB) | DNS | Service Mesh |
|---------|----------|----------|-----|--------------|
| **Layer** | Transport | Application | DNS | Application |
| **Speed** | Fastest | Fast | N/A | Fast |
| **Content Routing** | No | Yes | No | Yes |
| **SSL Termination** | Pass-through | Yes | No | Yes |
| **WebSocket** | Yes | Yes | N/A | Yes |
| **Health Checks** | TCP/HTTP | HTTP | Limited | HTTP |
| **Cost** | Lower | Higher | Lowest | Varies |
| **Use Case** | High throughput | Web apps | Simple distribution | Microservices |

---

## 1️⃣2️⃣ Interview Follow-Up Questions

### L4 (Junior/Mid) Level Questions

**Q1: What is the difference between L4 and L7 load balancing?**

**A**: 
- **L4 (Layer 4)**: Operates at transport layer (TCP/UDP). Makes routing decisions based on IP addresses and ports. Faster because it doesn't inspect content. Can't route based on URL or headers.
- **L7 (Layer 7)**: Operates at application layer (HTTP). Can inspect headers, URLs, cookies. Enables content-based routing (e.g., /api to API servers). Slower due to deeper inspection but more flexible.

**Q2: Explain the Round Robin algorithm.**

**A**: Round Robin distributes requests sequentially across servers in circular order. Request 1 goes to Server A, Request 2 to Server B, Request 3 to Server C, then back to Server A. Simple and effective for homogeneous servers with similar request patterns. Doesn't account for server capacity or current load.

**Q3: What are health checks and why are they important?**

**A**: Health checks are periodic probes to verify servers are functioning correctly. They're important because:
1. Automatically remove failed servers from rotation
2. Prevent sending traffic to unhealthy servers
3. Enable automatic recovery when servers come back online
4. Can check application health, not just network connectivity

### L5 (Senior) Level Questions

**Q4: How would you design a load balancing strategy for a global application?**

**A**: Multi-tier approach:

1. **DNS Level (GeoDNS)**:
   - Route users to nearest region
   - Use latency-based or geographic routing

2. **Edge Level (CDN/Global LB)**:
   - Cloudflare, AWS Global Accelerator
   - Anycast for automatic routing
   - DDoS protection

3. **Regional Level (L7 ALB)**:
   - Route to specific services
   - SSL termination
   - Content-based routing

4. **Service Level (Service Mesh)**:
   - Internal load balancing
   - Circuit breakers
   - Retries

```
User → GeoDNS → Regional Edge → ALB → Service Mesh → Pods
```

**Q5: How do you handle session persistence without sticky sessions?**

**A**: Multiple approaches:

1. **Externalized Sessions**: Store sessions in Redis/Memcached. Any server can handle any request.

2. **JWT Tokens**: Encode session data in signed tokens. No server-side session storage needed.

3. **Database Sessions**: Store sessions in database. Higher latency but durable.

4. **Client-Side Storage**: Store non-sensitive data in browser (localStorage, cookies).

Best practice is JWT for authentication + Redis for session data that changes frequently.

### L6 (Staff+) Level Questions

**Q6: Design a load balancing solution that handles 1 million requests per second.**

**A**: 
1. **Architecture**:
   - Multiple layers of load balancers
   - L4 (NLB) at edge for raw throughput
   - L7 (ALB) behind for routing
   - Anycast for geographic distribution

2. **Capacity Planning**:
   - Each L4 LB handles ~100K RPS
   - Need 10+ L4 LBs with DNS round-robin
   - Backend servers sized for 10K RPS each
   - 100+ backend servers per region

3. **Optimizations**:
   - Connection pooling to backends
   - Keep-alive connections
   - Efficient health checks (don't overwhelm backends)
   - Caching at edge (CDN)

4. **Resilience**:
   - Multi-AZ deployment
   - Auto-scaling groups
   - Circuit breakers
   - Graceful degradation

**Q7: A load balancer is causing intermittent 502 errors. How do you diagnose?**

**A**: Systematic debugging:

1. **Check LB metrics**:
   - Healthy host count (are backends dropping?)
   - Request count (traffic spike?)
   - Latency percentiles

2. **Check health check logs**:
   - Which servers are failing health checks?
   - What's the failure pattern?

3. **Check backend logs**:
   - Are backends returning errors?
   - Are they timing out?
   - Resource exhaustion (CPU, memory, connections)?

4. **Check network**:
   - Security group rules
   - Network ACLs
   - Connection limits

5. **Common causes**:
   - Backend timeout < LB timeout (LB thinks backend died)
   - Health check endpoint too slow
   - Backend connection pool exhausted
   - Keep-alive mismatch

```bash
# Useful commands
# Check ALB target health
aws elbv2 describe-target-health --target-group-arn <arn>

# Check backend connections
netstat -an | grep :8080 | wc -l

# Check for connection timeouts
tail -f /var/log/nginx/error.log | grep timeout
```

---

## 1️⃣3️⃣ Mental Summary

**Load balancers distribute traffic** across multiple servers to improve capacity, availability, and reliability. They're essential for any production system handling significant traffic.

**Key decisions**: L4 vs L7 (speed vs flexibility), algorithm choice (round robin for simple cases, least connections for varying load), sticky sessions (avoid if possible, use external sessions).

**Health checks are critical**: They automatically remove failed servers and add them back when recovered. Configure them carefully, not too aggressive (false positives) or too lenient (slow detection).

**For production**: Use managed load balancers (AWS ALB/NLB) when possible. Implement proper health checks. Externalize sessions. Plan for high availability (multiple LBs). Monitor metrics closely.

**For interviews**: Know the algorithms and their tradeoffs. Understand L4 vs L7. Be able to design multi-tier load balancing for global applications. Know how to debug common issues (502 errors, uneven distribution).

---

## 📚 Further Reading

- [AWS Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/)
- [Nginx Load Balancing Guide](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
- [HAProxy Documentation](https://www.haproxy.com/documentation/)
- [Google Maglev Paper](https://research.google/pubs/pub44824/)
- [Consistent Hashing Explained](https://www.toptal.com/big-data/consistent-hashing)

