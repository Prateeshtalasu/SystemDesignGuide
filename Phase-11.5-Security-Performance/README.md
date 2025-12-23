# 🔐 PHASE 11.5: SECURITY & PERFORMANCE (Week 16)

**Goal**: Build secure, fast systems

**Learning Objectives**:
- Implement authentication and authorization correctly
- Protect against common vulnerabilities
- Optimize system performance
- Design secure API architectures

**Estimated Time**: 15-20 hours

---

## Security Deep Dive:

### Transport Security

1. **HTTPS/TLS**
   - Certificate chain
   - TLS handshake flow
   - Cipher suites
   - Certificate management
   - Let's Encrypt automation
   - TLS versions (1.2 vs 1.3)

2. **mTLS (Mutual TLS)**
   - Certificate-based authentication
   - Service-to-service security
   - Certificate rotation
   - When to use mTLS
   - *Reference*: See Phase 2

### Authentication

3. **Authentication**
   - JWT structure (header, payload, signature)
   - JWT validation
   - Refresh tokens
   - Token expiration strategies
   - Token storage (cookies vs localStorage)
   - Session vs Token authentication

4. **OAuth2 & OpenID Connect**
   - OAuth2 flows (authorization code, client credentials, PKCE)
   - OpenID Connect (ID tokens)
   - Scopes and permissions
   - Token introspection
   - OAuth2 for microservices

### Authorization

5. **Authorization**
   - RBAC (Role-Based Access Control)
   - ABAC (Attribute-Based Access Control)
   - Policy-based authorization
   - Authorization patterns
   - Centralized vs distributed authorization

### Vulnerabilities

6. **Common Vulnerabilities (OWASP Top 10)**
   - SQL Injection (prepared statements)
   - XSS (Cross-Site Scripting) - escaping, CSP
   - CSRF (Cross-Site Request Forgery) - tokens
   - Clickjacking (X-Frame-Options)
   - SSRF (Server-Side Request Forgery)
   - Insecure deserialization
   - Broken authentication

7. **Input Validation**
   - Validation strategies
   - Sanitization
   - Allowlist vs blocklist
   - File upload security
   - API input validation

8. **Dependency Security**
   - Supply chain attacks
   - Dependency scanning (Snyk, Dependabot)
   - SBOM (Software Bill of Materials)
   - Vulnerability management
   - Secure dependency updates

### Secrets & Data

9. **Secrets Management**
   - Never commit secrets
   - AWS Secrets Manager
   - HashiCorp Vault
   - Environment variables
   - Secret rotation
   - Encryption at rest

10. **Data Protection**
    - Encryption at rest
    - Encryption in transit
    - PII handling
    - Data masking
    - GDPR/CCPA considerations

### API Security

11. **API Security Best Practices**
    - API key management
    - OAuth scopes
    - Rate limiting for security
    - API gateway security
    - Request signing
    - API versioning security

12. **Zero Trust Architecture**
    - Never trust, always verify
    - Micro-segmentation
    - Identity-based access
    - Continuous verification
    - Implementation strategies

---

## Performance Optimization:

### Connection Management

13. **Connection Pooling**
    - Database connections (HikariCP)
    - HTTP client connections
    - Why it matters (TCP overhead)
    - Pool sizing strategies
    - Connection leak detection

14. **Database Connection Pooling**
    - Pool sizing strategies
    - Connection timeout handling
    - Pool monitoring
    - HikariCP configuration
    - PgBouncer for PostgreSQL

### Query Optimization

15. **N+1 Query Problem**
    - Detection (query logging)
    - Solutions (JOIN, batch loading)
    - JPA fetch strategies
    - GraphQL DataLoader

16. **Database Query Optimization**
    - EXPLAIN plans
    - Index selection
    - Query rewriting
    - Denormalization
    - Read replicas

### Processing Patterns

17. **Batch Processing**
    - Bulk inserts
    - Batch updates
    - Trade-offs
    - Spring Batch basics

18. **Async Processing**
    - CompletableFuture
    - @Async in Spring
    - Event-driven patterns
    - When to go async
    - Async pitfalls

### Rate Limiting

19. **Rate Limiting Deep Dive**
    - Token bucket algorithm (detailed)
    - Leaky bucket algorithm
    - Sliding window log
    - Fixed window counter
    - Sliding window counter
    - Distributed rate limiting challenges
    - Rate limiting headers (X-RateLimit-*)

20. **API Throttling**
    - User-level throttling
    - API key-based throttling
    - Tiered rate limits
    - Burst vs sustained rate limits
    - Graceful degradation

### DDoS Protection

21. **DDoS Protection**
    - Types of DDoS attacks (volumetric, protocol, application)
    - Rate limiting as defense
    - IP whitelisting/blacklisting
    - CDN-based protection
    - Cloud provider DDoS protection (AWS Shield, CloudFlare)
    - WAF (Web Application Firewall)

### Compression & Caching

22. **Compression Algorithms**
    - gzip, deflate, brotli
    - When to compress (text vs binary)
    - Compression trade-offs (CPU vs bandwidth)
    - HTTP compression headers
    - Compression levels

23. **Caching Strategies for Performance**
    - HTTP caching (Cache-Control headers)
    - Browser caching
    - CDN caching
    - Application-level caching
    - Cache warming strategies
    - *Reference*: See Phase 4

### Profiling

24. **Performance Profiling**
    - CPU profiling
    - Memory profiling
    - Flame graphs
    - JProfiler, VisualVM, async-profiler
    - Continuous profiling in production

---

## Cross-References:
- **mTLS**: See Phase 2
- **Caching**: See Phase 4
- **Rate Limiting Patterns**: See Phase 5

---

## Practice Problems:
1. Implement JWT authentication with refresh tokens
2. Set up rate limiting with Redis
3. Optimize a slow database query
4. Perform a security audit on an API

---

## Common Interview Questions:
- "How do you prevent SQL injection?"
- "What's the difference between authentication and authorization?"
- "How would you handle a DDoS attack?"
- "How do you optimize a slow endpoint?"

---

## Deliverable
Can secure and optimize any system
