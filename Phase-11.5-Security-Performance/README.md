# 🔐 PHASE 11.5: SECURITY & PERFORMANCE (Week 16)

**Goal**: Build secure, fast systems

## Security Deep Dive:

1. **HTTPS/TLS**
   - Certificate chain
   - Handshake flow
   - Cipher suites

2. **Authentication**
   - JWT structure (header, payload, signature)
   - JWT validation
   - Refresh tokens
   - Token expiration strategies

3. **Authorization**
   - RBAC (Role-Based Access Control)
   - ABAC (Attribute-Based)
   - OAuth2 flow (authorization code, client credentials)
   - OpenID Connect

4. **Common Vulnerabilities**
   - SQL Injection (prepared statements)
   - XSS (Cross-Site Scripting) - escaping
   - CSRF (Cross-Site Request Forgery) - tokens
   - Clickjacking (X-Frame-Options)

5. **Secrets Management**
   - Never commit secrets
   - AWS Secrets Manager
   - HashiCorp Vault
   - Environment variables

## Performance Optimization:

1. **Connection Pooling**
   - Database connections (HikariCP)
   - HTTP client connections
   - Why it matters (TCP overhead)

2. **N+1 Query Problem**
   - Detection (query logging)
   - Solutions (JOIN, batch loading)
   - JPA fetch strategies

3. **Database Query Optimization**
   - EXPLAIN plans
   - Index selection
   - Denormalization
   - Read replicas

4. **Batch Processing**
   - Bulk inserts
   - Batch updates
   - Trade-offs

5. **Async Processing**
   - CompletableFuture
   - @Async in Spring
   - Event-driven patterns
   - When to go async

6. **Rate Limiting Deep Dive**
   - Token bucket algorithm (detailed)
   - Leaky bucket algorithm
   - Sliding window log
   - Fixed window counter
   - Distributed rate limiting challenges
   - Rate limiting headers (X-RateLimit-*)

7. **DDoS Protection**
   - Types of DDoS attacks
   - Rate limiting as defense
   - IP whitelisting/blacklisting
   - CDN-based protection
   - Cloud provider DDoS protection (AWS Shield)

8. **API Throttling**
   - User-level throttling
   - API key-based throttling
   - Tiered rate limits
   - Burst vs sustained rate limits

9. **Compression Algorithms**
   - gzip, deflate, brotli
   - When to compress (text vs binary)
   - Compression trade-offs (CPU vs bandwidth)
   - HTTP compression headers

10. **Database Connection Pooling**
    - Pool sizing strategies
    - Connection timeout handling
    - Pool monitoring
    - HikariCP configuration

11. **Caching Strategies for Performance**
    - HTTP caching (Cache-Control headers)
    - Browser caching
    - CDN caching
    - Application-level caching
    - Cache warming strategies

## Deliverable
Can secure and optimize any system

