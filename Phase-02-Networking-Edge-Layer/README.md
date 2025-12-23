# 🌐 PHASE 2: NETWORKING & EDGE LAYER (Week 2)

**Goal**: Understand how requests reach your system

**Learning Objectives**:
- Understand the complete request journey from client to server
- Master load balancing algorithms and their trade-offs
- Design edge layer for high-traffic systems
- Choose the right protocol for different use cases

**Estimated Time**: 15-20 hours

---

## 📚 Topics Overview

| # | Topic | File | Difficulty | Est. Time |
|---|-------|------|------------|-----------|
| 1 | DNS Deep Dive | `01-dns-deep-dive.md` | ⭐⭐ | 1.5 hrs |
| 2 | TCP vs UDP | `02-tcp-vs-udp.md` | ⭐⭐ | 1.5 hrs |
| 3 | HTTP Evolution | `03-http-evolution.md` | ⭐⭐⭐ | 1.5 hrs |
| 4 | Long Polling, WebSockets, SSE | `04-long-polling-websockets-sse.md` | ⭐⭐⭐ | 1.5 hrs |
| 5 | gRPC | `05-grpc.md` | ⭐⭐⭐ | 1.5 hrs |
| 6 | Load Balancers | `06-load-balancers.md` | ⭐⭐⭐ | 1.5 hrs |
| 7 | Reverse Proxy vs Forward Proxy | `07-reverse-proxy-forward-proxy.md` | ⭐⭐ | 1 hr |
| 8 | CDN | `08-cdn.md` | ⭐⭐⭐ | 1.5 hrs |
| 9 | API Gateway | `09-api-gateway.md` | ⭐⭐⭐ | 1.5 hrs |
| 10 | WebSockets Deep Dive | `10-websockets-deep-dive.md` | ⭐⭐⭐⭐ | 1.5 hrs |
| 11 | GraphQL | `11-graphql.md` | ⭐⭐⭐ | 1.5 hrs |
| 12 | API Design Principles | `12-api-design-principles.md` | ⭐⭐ | 1.5 hrs |
| 13 | Service Discovery | `13-service-discovery.md` | ⭐⭐⭐ | 1.5 hrs |
| 14 | mTLS | `14-mtls.md` | ⭐⭐⭐⭐ | 1.5 hrs |

---

## 🎯 Recommended Learning Path

### Week 2, Day 1-2: Foundations
Start with the fundamentals of how data travels across the internet.

```
1. DNS Deep Dive (01)
   └── Understand how domain names resolve to IPs
   
2. TCP vs UDP (02)
   └── Understand transport layer protocols
   
3. HTTP Evolution (03)
   └── Understand how web communication has evolved
```

### Week 2, Day 3-4: Communication Patterns
Learn different ways clients and servers communicate.

```
4. Long Polling, WebSockets, SSE (04)
   └── Real-time communication patterns
   
5. gRPC (05)
   └── High-performance RPC framework
   
10. WebSockets Deep Dive (10)
    └── Deep dive into bidirectional communication
```

### Week 2, Day 5-6: Edge Infrastructure
Understand the infrastructure that sits between clients and servers.

```
6. Load Balancers (06)
   └── Traffic distribution
   
7. Reverse Proxy vs Forward Proxy (07)
   └── Request intermediaries
   
8. CDN (08)
   └── Content delivery at the edge
   
9. API Gateway (09)
   └── API management and routing
```

### Week 2, Day 7: API Design & Security
Master API design and service security.

```
11. GraphQL (11)
    └── Alternative to REST APIs
    
12. API Design Principles (12)
    └── RESTful best practices
    
13. Service Discovery (13)
    └── Finding services in dynamic environments
    
14. mTLS (14)
    └── Secure service-to-service communication
```

---

## 🔗 Topic Dependencies

```
                    ┌─────────────┐
                    │  01. DNS    │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
       ┌───────────┐ ┌───────────┐ ┌───────────┐
       │ 02. TCP/  │ │ 06. Load  │ │ 08. CDN   │
       │    UDP    │ │ Balancers │ │           │
       └─────┬─────┘ └─────┬─────┘ └───────────┘
             │             │
             ▼             ▼
       ┌───────────┐ ┌───────────┐
       │ 03. HTTP  │ │ 07. Proxy │
       │ Evolution │ │           │
       └─────┬─────┘ └─────┬─────┘
             │             │
    ┌────────┼────────┐    │
    │        │        │    │
    ▼        ▼        ▼    ▼
┌───────┐ ┌───────┐ ┌───────────┐
│04. WS │ │05.gRPC│ │09. API    │
│SSE,LP │ │       │ │  Gateway  │
└───┬───┘ └───────┘ └───────────┘
    │
    ▼
┌───────────┐
│10. WS     │
│Deep Dive  │
└───────────┘

        ┌───────────┐
        │11.GraphQL │
        └─────┬─────┘
              │
              ▼
        ┌───────────┐     ┌───────────┐
        │12. API    │────▶│13.Service │
        │  Design   │     │ Discovery │
        └───────────┘     └─────┬─────┘
                                │
                                ▼
                          ┌───────────┐
                          │14. mTLS   │
                          └───────────┘
```

---

## 📖 Topics Detail

### 1. DNS Deep Dive
- Resolution flow (recursive vs iterative)
- TTL and caching
- DNS load balancing
- GeoDNS for global routing
- Record types (A, AAAA, CNAME, MX, TXT)

### 2. TCP vs UDP
- When to use each
- 3-way handshake
- Connection pooling
- TCP slow start and congestion control
- Java NIO for high-performance networking

### 3. HTTP Evolution
- HTTP/1.1 (keep-alive, pipelining)
- HTTP/2 (multiplexing, server push, HPACK compression)
- HTTP/3 (QUIC, UDP-based, 0-RTT)
- Head-of-line blocking and solutions

### 4. Long Polling vs WebSockets vs SSE
- Long Polling (HTTP-based, simple)
- Server-Sent Events (SSE) (one-way streaming)
- WebSockets (bidirectional, persistent)
- When to use each pattern
- Scaling with Redis Pub/Sub

### 5. gRPC
- vs REST comparison
- Protocol Buffers
- Streaming (unary, server, client, bidirectional)
- gRPC-Web for browsers
- Spring Boot integration

### 6. Load Balancers
- L4 (transport) vs L7 (application)
- Algorithms (Round Robin, Least Conn, IP Hash, Weighted)
- Health checks (active vs passive)
- Session persistence (sticky sessions)
- Connection draining

### 7. Reverse Proxy vs Forward Proxy
- Nginx configuration examples
- Use cases for each
- SSL termination
- Request buffering
- Caching at proxy layer

### 8. CDN (Content Delivery Network)
- Edge locations and PoPs
- Cache invalidation strategies
- Push vs Pull CDN
- CDN for dynamic content
- Multi-CDN strategies

### 9. API Gateway
- Authentication/Authorization
- Rate limiting
- Request routing
- Response transformation
- API composition
- Circuit breaker integration

### 10. WebSockets Deep Dive
- Connection lifecycle
- Heartbeat/ping-pong
- Frame types and structure
- Scaling WebSocket connections
- WebSocket vs Socket.io

### 11. GraphQL
- vs REST comparison
- Schema Definition Language (SDL)
- N+1 problem and DataLoader
- Federation for microservices
- Security considerations

### 12. API Design Principles
- RESTful best practices
- Resource naming conventions
- HTTP status codes (complete guide)
- API versioning strategies
- Pagination (cursor vs offset)
- HATEOAS

### 13. Service Discovery
- Client-side discovery
- Server-side discovery
- DNS-based discovery
- Eureka, Consul, Kubernetes comparison

### 14. mTLS (Mutual TLS)
- Certificate-based authentication
- Service-to-service security
- Certificate rotation strategies
- SPIFFE/SPIRE identity framework

---

## 🔄 Cross-References to Other Phases

| Topic | Related Phase | Connection |
|-------|---------------|------------|
| CDN Caching | Phase 4 (Caching) | Cache invalidation patterns |
| Service Discovery | Phase 10 (Microservices) | Deep dive into service mesh |
| Rate Limiting | Phase 5 (Distributed Systems) | Distributed rate limiting algorithms |
| API Security | Phase 11.5 (Security) | OAuth, JWT, security best practices |
| Load Balancing | Phase 10 (Microservices) | Service mesh load balancing |
| gRPC | Phase 10 (Microservices) | Inter-service communication |

---

## 💻 Practice Problems

### Beginner
1. Configure DNS records for a simple website
2. Implement a basic TCP echo server in Java
3. Make HTTP/2 requests using Java HttpClient

### Intermediate
4. Design the edge layer for a global e-commerce site
5. Choose between WebSocket, SSE, and Long Polling for a stock ticker
6. Configure Nginx as a reverse proxy with load balancing
7. Design API versioning strategy for a public API

### Advanced
8. Design a multi-region CDN strategy for video streaming
9. Implement a WebSocket server with Redis Pub/Sub for horizontal scaling
10. Design service discovery for a 1000+ microservice architecture

---

## 🎤 Common Interview Questions

### DNS & Networking
- "What happens when you type google.com in your browser?"
- "Explain DNS resolution step by step"
- "What is TTL and how does it affect DNS caching?"

### Load Balancing & Proxies
- "What's the difference between L4 and L7 load balancing?"
- "How do you handle sticky sessions with load balancers?"
- "What happens when a load balancer health check fails?"

### HTTP & Real-time Communication
- "What's the difference between HTTP/2 and HTTP/3?"
- "When would you use WebSockets vs SSE vs Long Polling?"
- "How do you scale WebSocket connections?"

### API Design
- "When would you use gRPC over REST?"
- "How does a CDN decide which edge server to route to?"
- "What is the N+1 problem in GraphQL and how do you solve it?"

### Security
- "What is mTLS and when would you use it?"
- "How do you handle certificate rotation in production?"

---

## ✅ Phase Completion Checklist

- [ ] Can explain DNS resolution flow end-to-end
- [ ] Understand when to use TCP vs UDP
- [ ] Know the differences between HTTP/1.1, HTTP/2, and HTTP/3
- [ ] Can choose the right real-time communication pattern
- [ ] Understand load balancer algorithms and trade-offs
- [ ] Can configure Nginx as reverse proxy
- [ ] Understand CDN caching strategies
- [ ] Can design an API Gateway architecture
- [ ] Know when to use GraphQL vs REST
- [ ] Understand service discovery patterns
- [ ] Can explain mTLS and certificate management

---

## 🎯 Deliverable

**Can design edge layer for Twitter-scale system**, including:
- Global DNS with GeoDNS routing
- Multi-tier load balancing (L4 + L7)
- CDN for static and dynamic content
- API Gateway with rate limiting and authentication
- WebSocket infrastructure for real-time features
- Service discovery for microservices
- mTLS for service-to-service security

---

## 📚 Additional Resources

### Books
- "High Performance Browser Networking" by Ilya Grigorik (free online)
- "TCP/IP Illustrated, Volume 1" by W. Richard Stevens

### Engineering Blogs
- [Cloudflare Blog](https://blog.cloudflare.com/) - Excellent networking content
- [Netflix Tech Blog](https://netflixtechblog.com/) - Real-world architecture
- [Discord Engineering](https://discord.com/blog) - WebSocket at scale

### Tools
- `dig` / `nslookup` - DNS debugging
- `curl` - HTTP testing
- `tcpdump` / Wireshark - Packet analysis
- `wrk` / `ab` - Load testing
