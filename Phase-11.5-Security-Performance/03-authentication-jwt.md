# 🔐 Authentication & JWT: Proving Identity

---

## 0️⃣ Prerequisites

Before diving into JWT authentication, you should understand:

- **HTTP Basics**: Request/response cycle, headers, cookies. Covered in Phase 2.
- **HTTPS/TLS**: Why transport encryption is essential. Covered in `01-https-tls.md`.
- **Base64 Encoding**: A way to represent binary data as ASCII text. Not encryption, just encoding.
- **Cryptographic Hashing**: One-way functions that produce fixed-size output (SHA-256). Covered in Phase 5.5.
- **Digital Signatures**: Using private keys to sign data, public keys to verify. Covered in `01-https-tls.md`.

**Quick Refresher on Base64**: Base64 converts binary data to text using 64 characters (A-Z, a-z, 0-9, +, /). It makes binary data safe to transmit in text-based protocols like HTTP. Base64URL is a variant that replaces + with - and / with _ to be URL-safe.

---

## 1️⃣ What Problem Does Authentication Exist to Solve?

### The Core Problem: HTTP is Stateless

HTTP was designed as a stateless protocol. Each request is independent with no memory of previous requests. But web applications need to remember users:

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    
    C->>S: Request 1: POST /login
    S->>C: "Welcome, Alice! Login successful."
    Note over S: Server has amnesia between requests!
    C->>S: Request 2: GET /my-account
    S->>C: "Who are you? I have no memory of you."
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE STATELESS PROBLEM                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Request 1: POST /login                                                  │
│  Server: "Welcome, Alice! Login successful."                             │
│                                                                          │
│  Request 2: GET /my-account                                              │
│  Server: "Who are you? I have no memory of you."                         │
│                                                                          │
│  The server has amnesia between requests!                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

</details>

### What Systems Looked Like Before Modern Auth

**Early Web (1990s)**: Basic Authentication
```
Authorization: Basic YWxpY2U6c2VjcmV0MTIz
                     └── base64("alice:secret123")
```
Problems:
- Password sent with EVERY request
- No logout mechanism
- Password stored in browser memory

**Session-Based Authentication (2000s)**:
```
1. User logs in with username/password
2. Server creates session, stores in database
3. Server sends session ID in cookie
4. Every request includes cookie
5. Server looks up session in database
```
Problems at scale:
- Database lookup on every request
- Session storage becomes bottleneck
- Sticky sessions or shared storage needed for multiple servers

### What Breaks Without Proper Authentication

| Problem | Impact |
|---------|--------|
| No authentication | Anyone can access anything |
| Weak authentication | Easy account takeover |
| Session fixation | Attacker hijacks sessions |
| No session expiration | Stolen tokens work forever |
| Insecure token storage | XSS leads to account compromise |

### The Scale Problem with Sessions

```mermaid
flowchart TD
    USERS["10 million users × 2 sessions each<br/>= 20 million session lookups"]
    
    SERVER1["Server 1"]
    SERVER2["Server 2"]
    SERVER3["Server 3"]
    
    STORE["Session Store<br/>(Redis/DB)<br/>20M entries<br/><br/>← Bottleneck!<br/>← Single point of failure!<br/>← Memory expensive!"]
    
    SOLUTION["JWT Solution:<br/>No central session store needed<br/>Each server can verify tokens independently"]
    
    SERVER1 --> STORE
    SERVER2 --> STORE
    SERVER3 --> STORE
    
    USERS --> SERVER1
    USERS --> SERVER2
    USERS --> SERVER3
    
    style USERS fill:#e3f2fd
    style SERVER1 fill:#fff9c4
    style SERVER2 fill:#fff9c4
    style SERVER3 fill:#fff9c4
    style STORE fill:#ffcdd2
    style SOLUTION fill:#c8e6c9
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    SESSION SCALING PROBLEM                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  10 million users × 2 sessions each = 20 million session lookups        │
│                                                                          │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐                           │
│  │ Server 1│     │ Server 2│     │ Server 3│                           │
│  └────┬────┘     └────┬────┘     └────┬────┘                           │
│       │               │               │                                  │
│       └───────────────┼───────────────┘                                 │
│                       │                                                  │
│                       ▼                                                  │
│              ┌────────────────┐                                         │
│              │ Session Store  │  ← Bottleneck!                          │
│              │ (Redis/DB)     │  ← Single point of failure!             │
│              │ 20M entries    │  ← Memory expensive!                    │
│              └────────────────┘                                         │
│                                                                          │
│  JWT Solution: No central session store needed                          │
│  Each server can verify tokens independently                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

</details>

---

## 2️⃣ Intuition and Mental Model

### JWT as a Stamped Passport

Think of a JWT like a passport:

**Traditional Session (Hotel Key Card)**:
- You get a plastic card with a number
- To enter your room, the door checks with the front desk
- Front desk looks up which room that card belongs to
- Every door needs to call the front desk

**JWT (Passport)**:
- Government issues you a passport with your info printed on it
- Passport has a special stamp (signature) that proves it's genuine
- Any border agent can verify the stamp without calling the government
- Your identity travels WITH you

```mermaid
flowchart TD
    subgraph JWT["PASSPORT (JWT)"]
        HEADER["HEADER (How to verify)<br/>Algorithm: RS256<br/>Type: JWT"]
        PAYLOAD["PAYLOAD (Who you are)<br/>Name: Alice Smith<br/>User ID: 12345<br/>Role: Premium User<br/>Issued: 2024-01-15<br/>Expires: 2024-01-15 + 1 hour"]
        SIGNATURE["SIGNATURE (Government Stamp)<br/>[Cryptographic proof this<br/>wasn't tampered with]<br/>[Only the issuer could<br/>have created this]"]
        
        HEADER --> PAYLOAD
        PAYLOAD --> SIGNATURE
    end
    
    NOTE["Any service can verify the signature<br/>without calling the issuer!"]
    
    JWT --> NOTE
    
    style JWT fill:#e3f2fd
    style HEADER fill:#fff9c4
    style PAYLOAD fill:#c8e6c9
    style SIGNATURE fill:#fce4ec
    style NOTE fill:#fff9c4
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                         JWT AS PASSPORT                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        PASSPORT (JWT)                            │    │
│  │  ┌─────────────────────────────────────────────────────────┐    │    │
│  │  │ HEADER (How to verify)                                   │    │    │
│  │  │ Algorithm: RS256                                         │    │    │
│  │  │ Type: JWT                                                │    │    │
│  │  └─────────────────────────────────────────────────────────┘    │    │
│  │  ┌─────────────────────────────────────────────────────────┐    │    │
│  │  │ PAYLOAD (Who you are)                                    │    │    │
│  │  │ Name: Alice Smith                                        │    │    │
│  │  │ User ID: 12345                                           │    │    │
│  │  │ Role: Premium User                                       │    │    │
│  │  │ Issued: 2024-01-15                                       │    │    │
│  │  │ Expires: 2024-01-15 + 1 hour                             │    │    │
│  │  └─────────────────────────────────────────────────────────┘    │    │
│  │  ┌─────────────────────────────────────────────────────────┐    │    │
│  │  │ SIGNATURE (Government Stamp)                             │    │    │
│  │  │ [Cryptographic proof this wasn't tampered with]          │    │    │
│  │  │ [Only the issuer could have created this]                │    │    │
│  │  └─────────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Any service can verify the signature without calling the issuer!       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

</details>

---

## 3️⃣ How JWT Works Internally

### JWT Structure

A JWT consists of three parts separated by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkFsaWNlIn0.signature
└──────────── Header ────────────┘.└────────────── Payload ──────────────┘.└─ Signature ─┘
```

**Header** (Base64URL encoded JSON):
```json
{
  "alg": "HS256",    // Algorithm used for signature
  "typ": "JWT"       // Type of token
}
```

**Payload** (Base64URL encoded JSON):
```json
{
  "sub": "1234567890",           // Subject (user ID)
  "name": "Alice",               // Custom claim
  "iat": 1516239022,             // Issued At (Unix timestamp)
  "exp": 1516242622,             // Expiration (Unix timestamp)
  "iss": "https://auth.example.com",  // Issuer
  "aud": "https://api.example.com"    // Audience
}
```

**Signature** (depends on algorithm):
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

### Standard JWT Claims

| Claim | Name | Purpose |
|-------|------|---------|
| `iss` | Issuer | Who created the token |
| `sub` | Subject | Who the token is about (user ID) |
| `aud` | Audience | Who the token is intended for |
| `exp` | Expiration | When the token expires |
| `nbf` | Not Before | Token not valid before this time |
| `iat` | Issued At | When the token was created |
| `jti` | JWT ID | Unique identifier for the token |

### Signing Algorithms

**Symmetric (Shared Secret)**:
- `HS256`, `HS384`, `HS512` (HMAC with SHA)
- Same secret for signing and verifying
- Simpler but secret must be shared

**Asymmetric (Public/Private Key)**:
- `RS256`, `RS384`, `RS512` (RSA)
- `ES256`, `ES384`, `ES512` (ECDSA)
- `PS256`, `PS384`, `PS512` (RSA-PSS)
- Private key signs, public key verifies
- Better for distributed systems

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SYMMETRIC vs ASYMMETRIC                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  SYMMETRIC (HS256)                                                       │
│  ┌──────────────┐         ┌──────────────┐                              │
│  │ Auth Server  │ SECRET  │ API Server   │                              │
│  │              │ ═══════ │              │                              │
│  │ Signs with   │         │ Verifies with│                              │
│  │ SECRET       │         │ SAME SECRET  │                              │
│  └──────────────┘         └──────────────┘                              │
│  Problem: All servers need the secret                                    │
│                                                                          │
│  ASYMMETRIC (RS256)                                                      │
│  ┌──────────────┐         ┌──────────────┐                              │
│  │ Auth Server  │         │ API Server   │                              │
│  │              │         │              │                              │
│  │ Signs with   │ PUBLIC  │ Verifies with│                              │
│  │ PRIVATE KEY  │ ──────> │ PUBLIC KEY   │                              │
│  │ (kept secret)│         │ (can share)  │                              │
│  └──────────────┘         └──────────────┘                              │
│  Better: Only auth server has private key                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Token Verification Process

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    JWT VERIFICATION STEPS                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. SPLIT TOKEN                                                          │
│     "header.payload.signature" → [header, payload, signature]           │
│                                                                          │
│  2. DECODE HEADER                                                        │
│     Base64URL decode → {"alg": "RS256", "typ": "JWT"}                   │
│     Check algorithm is allowed (prevent algorithm confusion attack)      │
│                                                                          │
│  3. VERIFY SIGNATURE                                                     │
│     Recompute: signature' = sign(header + "." + payload, key)           │
│     Compare: signature' == signature?                                    │
│     If no match → REJECT (tampered or wrong key)                        │
│                                                                          │
│  4. DECODE PAYLOAD                                                       │
│     Base64URL decode → claims object                                     │
│                                                                          │
│  5. VALIDATE CLAIMS                                                      │
│     • exp: Is current time < expiration? (not expired)                  │
│     • nbf: Is current time > not-before? (already valid)                │
│     • iss: Is issuer who we expect?                                     │
│     • aud: Are we the intended audience?                                │
│                                                                          │
│  6. EXTRACT USER INFO                                                    │
│     Use sub, roles, permissions from payload                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Simulation-First Explanation

### Complete Authentication Flow

Let's trace a complete login and API access flow:

**Step 1: User Logs In**
```http
POST /auth/login HTTP/1.1
Host: auth.example.com
Content-Type: application/json

{
  "email": "alice@example.com",
  "password": "SecureP@ss123"
}
```

**Step 2: Server Validates Credentials**
```java
// Server-side pseudocode
User user = userRepository.findByEmail("alice@example.com");
if (user == null) {
    return 401; // User not found
}
if (!passwordEncoder.matches("SecureP@ss123", user.getPasswordHash())) {
    return 401; // Wrong password
}
// Credentials valid, create tokens
```

**Step 3: Server Creates JWT**
```java
// Create access token (short-lived)
String accessToken = Jwts.builder()
    .setSubject(user.getId())                    // sub: "user-123"
    .setIssuer("https://auth.example.com")       // iss
    .setAudience("https://api.example.com")      // aud
    .setIssuedAt(new Date())                     // iat
    .setExpiration(Date.from(Instant.now()
        .plus(15, ChronoUnit.MINUTES)))          // exp: 15 minutes
    .claim("roles", user.getRoles())             // custom claim
    .claim("email", user.getEmail())             // custom claim
    .signWith(privateKey, SignatureAlgorithm.RS256)
    .compact();

// Create refresh token (long-lived)
String refreshToken = Jwts.builder()
    .setSubject(user.getId())
    .setIssuedAt(new Date())
    .setExpiration(Date.from(Instant.now()
        .plus(7, ChronoUnit.DAYS)))              // exp: 7 days
    .claim("type", "refresh")
    .signWith(privateKey, SignatureAlgorithm.RS256)
    .compact();
```

**Step 4: Server Returns Tokens**
```http
HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: refreshToken=eyJ...; HttpOnly; Secure; SameSite=Strict; Path=/auth

{
  "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyLTEyMyIsImlzcyI6Imh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbSIsImF1ZCI6Imh0dHBzOi8vYXBpLmV4YW1wbGUuY29tIiwiaWF0IjoxNzA1MzEyMDAwLCJleHAiOjE3MDUzMTI5MDAsInJvbGVzIjpbInVzZXIiXSwiZW1haWwiOiJhbGljZUBleGFtcGxlLmNvbSJ9.signature",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

**Step 5: Client Stores Tokens**
```javascript
// Frontend code
// Store access token in memory (not localStorage!)
let accessToken = response.accessToken;

// Refresh token is in HttpOnly cookie (automatic)
```

**Step 6: Client Makes API Request**
```http
GET /api/v1/orders HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

**Step 7: API Server Validates Token**
```java
// API server pseudocode
String token = extractBearerToken(request);

try {
    Claims claims = Jwts.parserBuilder()
        .setSigningKey(publicKey)           // Only need public key!
        .requireIssuer("https://auth.example.com")
        .requireAudience("https://api.example.com")
        .build()
        .parseClaimsJws(token)
        .getBody();
    
    String userId = claims.getSubject();
    List<String> roles = claims.get("roles", List.class);
    
    // User authenticated! Proceed with request
    return orderService.getOrdersForUser(userId);
    
} catch (ExpiredJwtException e) {
    return 401; // Token expired
} catch (JwtException e) {
    return 401; // Invalid token
}
```

### Token Refresh Flow

When the access token expires:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TOKEN REFRESH FLOW                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. API call fails with 401 (token expired)                             │
│                                                                          │
│  2. Client sends refresh request:                                        │
│     POST /auth/refresh                                                   │
│     Cookie: refreshToken=eyJ...                                          │
│                                                                          │
│  3. Auth server validates refresh token:                                │
│     • Is signature valid?                                                │
│     • Is token expired?                                                  │
│     • Is token revoked? (check blacklist)                               │
│     • Is token type "refresh"?                                          │
│                                                                          │
│  4. Auth server issues new tokens:                                       │
│     • New access token (15 min)                                          │
│     • Optionally new refresh token (rotation)                           │
│                                                                          │
│  5. Client retries original request with new access token               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Netflix's Token Architecture

Netflix uses a sophisticated token system:
- **Short-lived access tokens**: 10-15 minutes
- **Device-bound tokens**: Tied to device fingerprint
- **Hierarchical tokens**: Different tokens for different service tiers
- **Token introspection**: For high-security operations

### Auth0's Best Practices

Auth0 (identity provider used by many companies) recommends:
- Access tokens: 1-24 hours depending on sensitivity
- Refresh tokens: 2 weeks to 6 months
- Refresh token rotation: Issue new refresh token on each use
- Absolute timeout: Force re-authentication after X days

### Token Storage Best Practices

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TOKEN STORAGE COMPARISON                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  STORAGE OPTION        SECURITY           USE CASE                      │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  localStorage          ❌ XSS vulnerable   NEVER for tokens              │
│  sessionStorage        ❌ XSS vulnerable   NEVER for tokens              │
│                                                                          │
│  Memory (JS variable)  ✅ XSS safer        Access tokens                 │
│                        ⚠️ Lost on refresh  (short-lived)                 │
│                                                                          │
│  HttpOnly Cookie       ✅ XSS immune       Refresh tokens                │
│                        ⚠️ CSRF risk        (with CSRF protection)        │
│                                                                          │
│  RECOMMENDED PATTERN:                                                    │
│  • Access token: In-memory JavaScript variable                          │
│  • Refresh token: HttpOnly, Secure, SameSite=Strict cookie             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6️⃣ How to Implement JWT Authentication in Java

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Spring Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    
    <!-- JWT Library (JJWT) -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.3</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### JWT Service Implementation

```java
package com.example.security.jwt;

import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.Date;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * Service for creating and validating JWTs.
 * 
 * This implementation uses HMAC (symmetric) for simplicity.
 * For production with multiple services, use RSA (asymmetric).
 */
@Service
public class JwtService {

    private final SecretKey secretKey;
    private final long accessTokenExpirationMinutes;
    private final long refreshTokenExpirationDays;
    private final String issuer;

    public JwtService(
            @Value("${jwt.secret}") String secret,
            @Value("${jwt.access-token-expiration-minutes:15}") long accessTokenExpiration,
            @Value("${jwt.refresh-token-expiration-days:7}") long refreshTokenExpiration,
            @Value("${jwt.issuer:https://auth.example.com}") String issuer) {
        
        // Secret must be at least 256 bits (32 bytes) for HS256
        if (secret.length() < 32) {
            throw new IllegalArgumentException("JWT secret must be at least 32 characters");
        }
        
        this.secretKey = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.accessTokenExpirationMinutes = accessTokenExpiration;
        this.refreshTokenExpirationDays = refreshTokenExpiration;
        this.issuer = issuer;
    }

    /**
     * Creates an access token for a user.
     * 
     * Access tokens are short-lived and contain user claims.
     * They're sent with every API request.
     */
    public String createAccessToken(String userId, String email, List<String> roles) {
        Instant now = Instant.now();
        Instant expiration = now.plus(accessTokenExpirationMinutes, ChronoUnit.MINUTES);
        
        return Jwts.builder()
            // Standard claims
            .subject(userId)                              // Who this token is about
            .issuer(issuer)                               // Who issued the token
            .issuedAt(Date.from(now))                     // When it was issued
            .expiration(Date.from(expiration))            // When it expires
            .id(UUID.randomUUID().toString())             // Unique token ID
            
            // Custom claims
            .claim("email", email)
            .claim("roles", roles)
            .claim("type", "access")
            
            // Sign the token
            .signWith(secretKey, Jwts.SIG.HS256)
            .compact();
    }

    /**
     * Creates a refresh token for a user.
     * 
     * Refresh tokens are long-lived and stored securely (HttpOnly cookie).
     * They're only used to get new access tokens.
     */
    public String createRefreshToken(String userId) {
        Instant now = Instant.now();
        Instant expiration = now.plus(refreshTokenExpirationDays, ChronoUnit.DAYS);
        
        return Jwts.builder()
            .subject(userId)
            .issuer(issuer)
            .issuedAt(Date.from(now))
            .expiration(Date.from(expiration))
            .id(UUID.randomUUID().toString())
            .claim("type", "refresh")
            .signWith(secretKey, Jwts.SIG.HS256)
            .compact();
    }

    /**
     * Validates a token and returns its claims.
     * 
     * This method:
     * 1. Verifies the signature (proves token wasn't tampered with)
     * 2. Checks expiration (token not expired)
     * 3. Validates issuer (token from trusted source)
     * 
     * @throws JwtException if token is invalid
     */
    public Claims validateToken(String token) {
        return Jwts.parser()
            .verifyWith(secretKey)
            .requireIssuer(issuer)
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    /**
     * Validates and extracts user info from an access token.
     */
    public TokenUserInfo validateAccessToken(String token) {
        Claims claims = validateToken(token);
        
        // Verify this is an access token, not a refresh token
        String tokenType = claims.get("type", String.class);
        if (!"access".equals(tokenType)) {
            throw new JwtException("Invalid token type");
        }
        
        return new TokenUserInfo(
            claims.getSubject(),
            claims.get("email", String.class),
            claims.get("roles", List.class)
        );
    }

    /**
     * Validates a refresh token and returns the user ID.
     */
    public String validateRefreshToken(String token) {
        Claims claims = validateToken(token);
        
        String tokenType = claims.get("type", String.class);
        if (!"refresh".equals(tokenType)) {
            throw new JwtException("Invalid token type");
        }
        
        return claims.getSubject();
    }

    /**
     * Extracts user info from token without validation.
     * ONLY use this for non-security purposes (like logging).
     */
    public TokenUserInfo extractUserInfoUnsafe(String token) {
        // Split token and decode payload (middle part)
        String[] parts = token.split("\\.");
        if (parts.length != 3) {
            throw new IllegalArgumentException("Invalid JWT format");
        }
        
        // This does NOT verify the signature!
        Claims claims = Jwts.parser()
            .unsecured()
            .build()
            .parseUnsecuredClaims(parts[0] + "." + parts[1] + ".")
            .getPayload();
        
        return new TokenUserInfo(
            claims.getSubject(),
            claims.get("email", String.class),
            claims.get("roles", List.class)
        );
    }

    /**
     * Record to hold extracted user info.
     */
    public record TokenUserInfo(String userId, String email, List<String> roles) {}
}
```

### Spring Security Configuration

```java
package com.example.security.config;

import com.example.security.jwt.JwtAuthenticationFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

/**
 * Spring Security configuration for JWT authentication.
 * 
 * Key points:
 * - Stateless session (no server-side session)
 * - JWT filter runs before UsernamePasswordAuthenticationFilter
 * - Public endpoints don't require authentication
 */
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // Enables @PreAuthorize annotations
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthFilter) {
        this.jwtAuthFilter = jwtAuthFilter;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Disable CSRF for stateless API (tokens provide protection)
            .csrf(csrf -> csrf.disable())
            
            // Stateless session - no session cookies
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            
            // Authorization rules
            .authorizeHttpRequests(auth -> auth
                // Public endpoints
                .requestMatchers("/auth/**").permitAll()
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/health", "/metrics").permitAll()
                
                // Admin endpoints require ADMIN role
                .requestMatchers("/admin/**").hasRole("ADMIN")
                
                // Everything else requires authentication
                .anyRequest().authenticated()
            )
            
            // Add JWT filter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        // BCrypt with strength 12 (2^12 iterations)
        return new BCryptPasswordEncoder(12);
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

### JWT Authentication Filter

```java
package com.example.security.jwt;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

/**
 * Filter that validates JWT tokens on every request.
 * 
 * This filter:
 * 1. Extracts token from Authorization header
 * 2. Validates the token
 * 3. Sets up Spring Security context with user info
 */
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;

    public JwtAuthenticationFilter(JwtService jwtService) {
        this.jwtService = jwtService;
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {
        
        // Extract token from header
        String authHeader = request.getHeader("Authorization");
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            // No token provided, continue to next filter
            // Spring Security will handle unauthorized access
            filterChain.doFilter(request, response);
            return;
        }
        
        String token = authHeader.substring(7); // Remove "Bearer " prefix
        
        try {
            // Validate token and extract user info
            JwtService.TokenUserInfo userInfo = jwtService.validateAccessToken(token);
            
            // Convert roles to Spring Security authorities
            List<SimpleGrantedAuthority> authorities = userInfo.roles().stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .toList();
            
            // Create authentication object
            UsernamePasswordAuthenticationToken authentication = 
                new UsernamePasswordAuthenticationToken(
                    userInfo.userId(),  // Principal (user identifier)
                    null,               // Credentials (not needed, token already validated)
                    authorities         // Granted authorities (roles)
                );
            
            // Store user info for later use
            authentication.setDetails(userInfo);
            
            // Set authentication in security context
            SecurityContextHolder.getContext().setAuthentication(authentication);
            
        } catch (Exception e) {
            // Token validation failed
            // Clear any existing authentication
            SecurityContextHolder.clearContext();
            
            // Log the error (but don't expose details to client)
            logger.debug("JWT validation failed: " + e.getMessage());
        }
        
        filterChain.doFilter(request, response);
    }
}
```

### Authentication Controller

```java
package com.example.security.controller;

import com.example.security.jwt.JwtService;
import com.example.security.service.UserService;
import jakarta.servlet.http.Cookie;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.validation.Valid;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

/**
 * Authentication endpoints for login, logout, and token refresh.
 */
@RestController
@RequestMapping("/auth")
public class AuthController {

    private final UserService userService;
    private final JwtService jwtService;

    public AuthController(UserService userService, JwtService jwtService) {
        this.userService = userService;
        this.jwtService = jwtService;
    }

    /**
     * Login endpoint - validates credentials and returns tokens.
     */
    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(
            @Valid @RequestBody LoginRequest request,
            HttpServletResponse response) {
        
        // Validate credentials
        User user = userService.authenticate(request.email(), request.password());
        if (user == null) {
            return ResponseEntity.status(401).build();
        }
        
        // Create tokens
        String accessToken = jwtService.createAccessToken(
            user.getId(), 
            user.getEmail(), 
            user.getRoles()
        );
        String refreshToken = jwtService.createRefreshToken(user.getId());
        
        // Set refresh token as HttpOnly cookie
        Cookie refreshCookie = new Cookie("refreshToken", refreshToken);
        refreshCookie.setHttpOnly(true);    // Not accessible via JavaScript
        refreshCookie.setSecure(true);      // Only sent over HTTPS
        refreshCookie.setPath("/auth");     // Only sent to /auth endpoints
        refreshCookie.setMaxAge(7 * 24 * 60 * 60);  // 7 days
        refreshCookie.setAttribute("SameSite", "Strict");  // CSRF protection
        response.addCookie(refreshCookie);
        
        return ResponseEntity.ok(new LoginResponse(
            accessToken,
            "Bearer",
            900  // 15 minutes in seconds
        ));
    }

    /**
     * Refresh endpoint - exchanges refresh token for new access token.
     */
    @PostMapping("/refresh")
    public ResponseEntity<LoginResponse> refresh(
            @CookieValue(name = "refreshToken", required = false) String refreshToken,
            HttpServletResponse response) {
        
        if (refreshToken == null) {
            return ResponseEntity.status(401).build();
        }
        
        try {
            // Validate refresh token
            String userId = jwtService.validateRefreshToken(refreshToken);
            
            // Get user (to get current roles, email)
            User user = userService.findById(userId);
            if (user == null) {
                return ResponseEntity.status(401).build();
            }
            
            // Create new access token
            String newAccessToken = jwtService.createAccessToken(
                user.getId(),
                user.getEmail(),
                user.getRoles()
            );
            
            // Optionally rotate refresh token (more secure)
            String newRefreshToken = jwtService.createRefreshToken(user.getId());
            Cookie refreshCookie = new Cookie("refreshToken", newRefreshToken);
            refreshCookie.setHttpOnly(true);
            refreshCookie.setSecure(true);
            refreshCookie.setPath("/auth");
            refreshCookie.setMaxAge(7 * 24 * 60 * 60);
            refreshCookie.setAttribute("SameSite", "Strict");
            response.addCookie(refreshCookie);
            
            return ResponseEntity.ok(new LoginResponse(
                newAccessToken,
                "Bearer",
                900
            ));
            
        } catch (Exception e) {
            return ResponseEntity.status(401).build();
        }
    }

    /**
     * Logout endpoint - clears refresh token cookie.
     */
    @PostMapping("/logout")
    public ResponseEntity<Void> logout(HttpServletResponse response) {
        // Clear the refresh token cookie
        Cookie refreshCookie = new Cookie("refreshToken", "");
        refreshCookie.setHttpOnly(true);
        refreshCookie.setSecure(true);
        refreshCookie.setPath("/auth");
        refreshCookie.setMaxAge(0);  // Delete cookie
        response.addCookie(refreshCookie);
        
        return ResponseEntity.ok().build();
    }

    // Request/Response records
    public record LoginRequest(
        @NotBlank @Email String email,
        @NotBlank @Size(min = 8) String password
    ) {}
    
    public record LoginResponse(
        String accessToken,
        String tokenType,
        int expiresIn
    ) {}
}
```

### Application Configuration

```yaml
# application.yml
jwt:
  # IMPORTANT: In production, use environment variable or secrets manager
  # This should be at least 32 characters for HS256
  secret: ${JWT_SECRET:your-256-bit-secret-key-here-min-32-chars}
  access-token-expiration-minutes: 15
  refresh-token-expiration-days: 7
  issuer: https://auth.example.com

spring:
  security:
    # Disable default security (we're using custom JWT auth)
    user:
      name: disabled
      password: disabled
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common JWT Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Storing tokens in localStorage | XSS can steal tokens | Use memory + HttpOnly cookies |
| Long-lived access tokens | Extended exposure if stolen | Short expiration (15 min) |
| No token expiration | Tokens work forever | Always set `exp` claim |
| Weak secret key | Tokens can be forged | Use 256+ bit random secret |
| Algorithm confusion | Attacker can forge tokens | Explicitly verify algorithm |
| Sensitive data in payload | Anyone can decode payload | Only put non-sensitive claims |
| No refresh token rotation | Stolen refresh = permanent access | Rotate on each use |
| Not validating issuer/audience | Accept tokens from anywhere | Always validate `iss` and `aud` |

### The Algorithm Confusion Attack

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ALGORITHM CONFUSION ATTACK                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Server expects RS256 (asymmetric):                                      │
│  • Sign with PRIVATE key                                                 │
│  • Verify with PUBLIC key                                                │
│                                                                          │
│  Attacker sends token with alg: "HS256" (symmetric):                    │
│  • Signs with the PUBLIC key (which is publicly available)              │
│  • Server uses PUBLIC key for verification                              │
│  • Signature matches! Token accepted!                                    │
│                                                                          │
│  PREVENTION:                                                             │
│  • Never trust the alg header blindly                                   │
│  • Explicitly specify expected algorithm in verification                │
│  • Use libraries that require algorithm specification                   │
│                                                                          │
│  // WRONG - trusts alg from token                                        │
│  jwt.verify(token, publicKey);                                          │
│                                                                          │
│  // RIGHT - explicitly specifies algorithm                               │
│  jwt.verify(token, publicKey, { algorithms: ['RS256'] });               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### JWT vs Session Tradeoffs

| Aspect | JWT | Sessions |
|--------|-----|----------|
| Scalability | ✅ No central store | ❌ Requires shared store |
| Revocation | ❌ Hard (need blacklist) | ✅ Easy (delete from store) |
| Size | ❌ Larger (contains claims) | ✅ Small (just ID) |
| Stateless | ✅ Yes | ❌ No |
| Offline validation | ✅ Yes | ❌ No |
| Real-time updates | ❌ Wait for expiration | ✅ Immediate |

---

## 8️⃣ When NOT to Use JWT

### Cases Where Sessions Are Better

1. **Need instant revocation**: If you must immediately invalidate tokens (e.g., user changes password), sessions are simpler.

2. **Small scale**: For simple apps with few users, session overhead is negligible.

3. **Sensitive operations**: For banking/healthcare, session revocation is critical.

4. **Large payloads**: If you need many claims, JWT size becomes problematic.

### Hybrid Approach

Many systems use both:
- JWT for API authentication (stateless, scalable)
- Session-like behavior via short expiration + refresh tokens
- Blacklist for compromised tokens (checked only for high-security operations)

---

## 9️⃣ Comparison with Alternatives

### JWT vs Opaque Tokens

| Aspect | JWT | Opaque Tokens |
|--------|-----|---------------|
| Self-contained | Yes (contains claims) | No (just random string) |
| Validation | Local (signature check) | Remote (introspection endpoint) |
| Size | Larger | Smaller |
| Revocation | Hard | Easy |
| Privacy | Claims visible (base64) | Claims hidden |

### JWT vs API Keys

| Aspect | JWT | API Keys |
|--------|-----|----------|
| Expiration | Built-in | Manual tracking |
| User context | Contains user info | Requires lookup |
| Rotation | Automatic (short-lived) | Manual |
| Granularity | Per-session | Per-application |

---

## 🔟 Interview Follow-Up Questions

### L4 (Entry Level) Questions

**Q: What is a JWT and what are its three parts?**
A: JWT (JSON Web Token) is a compact, URL-safe token format for securely transmitting information. It has three parts separated by dots: Header (specifies algorithm and token type), Payload (contains claims like user ID, roles, expiration), and Signature (cryptographic proof the token wasn't tampered with). Each part is Base64URL encoded.

**Q: Where should you store JWTs on the client side?**
A: Access tokens should be stored in memory (JavaScript variable), not localStorage or sessionStorage, because those are vulnerable to XSS attacks. Refresh tokens should be stored in HttpOnly cookies, which JavaScript can't access. The cookie should also have Secure (HTTPS only) and SameSite=Strict flags.

### L5 (Mid Level) Questions

**Q: How do you handle JWT revocation?**
A: JWTs are stateless, so revocation is challenging. Options include:
1. Short expiration times (15 minutes) so stolen tokens quickly become useless
2. Blacklist/denylist of revoked token IDs (checked on each request)
3. Version number in token, increment on logout (requires lookup)
4. Refresh token rotation, invalidate old refresh tokens
The best approach is short access tokens + refresh token rotation + blacklist for critical operations.

**Q: Explain the refresh token flow and why it's needed.**
A: Access tokens are short-lived (15 minutes) for security. When they expire, instead of making users log in again, we use refresh tokens. The flow:
1. Access token expires
2. Client sends refresh token to auth server
3. Auth server validates refresh token, issues new access token
4. Client continues with new access token
Refresh tokens are long-lived but stored securely (HttpOnly cookie) and can be rotated/revoked. This balances security (short access token exposure) with UX (users stay logged in).

### L6 (Senior) Questions

**Q: Design a token-based auth system for a microservices architecture with 50 services.**
A:
1. **Centralized Auth Service**: Issues and validates tokens, manages refresh tokens
2. **Asymmetric keys (RS256)**: Auth service has private key, all services have public key
3. **Token structure**: Short access tokens (10 min), refresh tokens (7 days)
4. **JWKS endpoint**: Services fetch public keys from well-known URL, cache with TTL
5. **Token introspection**: For high-security operations, call auth service to verify
6. **Blacklist**: Redis-based blacklist for revoked tokens, checked for sensitive operations
7. **Service-to-service**: Separate service tokens with machine identity
8. **Monitoring**: Track token issuance rate, validation failures, suspicious patterns

**Q: How would you migrate from session-based to JWT-based authentication without downtime?**
A:
1. **Phase 1**: Deploy JWT infrastructure alongside sessions
2. **Phase 2**: New logins get both session and JWT, APIs accept both
3. **Phase 3**: Migrate clients to use JWT, monitor adoption
4. **Phase 4**: Set deadline, force re-login for remaining session users
5. **Phase 5**: Remove session infrastructure
Key considerations: backward compatibility, gradual rollout, monitoring, rollback plan

---

## 1️⃣1️⃣ One Clean Mental Summary

JWT authentication solves the stateless HTTP problem by embedding user identity directly in a cryptographically signed token. Instead of storing sessions on the server and looking them up on every request, the server can verify the token's signature locally. This makes JWTs perfect for distributed systems where you don't want a central session store. The tradeoff is that revocation is harder, which is why we use short-lived access tokens (minutes) paired with long-lived refresh tokens (days). Store access tokens in memory, refresh tokens in HttpOnly cookies, and always validate the algorithm, issuer, audience, and expiration.

---

## Quick Reference

```java
// Create token
String token = Jwts.builder()
    .subject(userId)
    .expiration(Date.from(Instant.now().plus(15, ChronoUnit.MINUTES)))
    .signWith(key)
    .compact();

// Validate token
Claims claims = Jwts.parser()
    .verifyWith(key)
    .build()
    .parseSignedClaims(token)
    .getPayload();
```

```bash
# Decode JWT (without verification)
echo "eyJhbG..." | cut -d'.' -f2 | base64 -d | jq

# Generate secure secret
openssl rand -base64 32
```

---

## References

- [JWT.io](https://jwt.io/) - JWT debugger and library list
- [RFC 7519](https://tools.ietf.org/html/rfc7519) - JWT specification
- [OWASP JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [Auth0 JWT Handbook](https://auth0.com/resources/ebooks/jwt-handbook)

