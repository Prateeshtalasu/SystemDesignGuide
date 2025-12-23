# 🔐 OAuth2 & OpenID Connect: Delegated Authorization and Identity

---

## 0️⃣ Prerequisites

Before diving into OAuth2 and OpenID Connect, you should understand:

- **JWT (JSON Web Tokens)**: Token format used by many OAuth2/OIDC implementations. Covered in `03-authentication-jwt.md`.
- **HTTPS/TLS**: All OAuth2 flows require secure transport. Covered in `01-https-tls.md`.
- **HTTP Redirects**: How 302 redirects work and how browsers handle them.
- **Authentication vs Authorization**: Authentication proves WHO you are. Authorization determines WHAT you can access.

**Quick Refresher**: Authentication answers "Who are you?" (identity). Authorization answers "What can you do?" (permissions). OAuth2 is primarily about authorization (delegating access), while OpenID Connect adds authentication (proving identity) on top of OAuth2.

---

## 1️⃣ What Problem Does OAuth2 Exist to Solve?

### The Core Problem: Sharing Access Without Sharing Passwords

Imagine you want a third-party app (like a photo printing service) to access your photos stored on Google. Before OAuth2:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE PASSWORD SHARING PROBLEM                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  BEFORE OAUTH2:                                                          │
│                                                                          │
│  User: "I want PrintMyPhotos.com to print my Google Photos"              │
│                                                                          │
│  PrintMyPhotos: "Give me your Google username and password"              │
│                                                                          │
│  Problems:                                                                │
│  ❌ PrintMyPhotos now has FULL access to your Google account            │
│  ❌ They could read your email, change your password, etc.              │
│  ❌ You can't revoke access without changing your password              │
│  ❌ If PrintMyPhotos is hacked, your Google account is compromised      │
│  ❌ You're training users to enter passwords on third-party sites       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### What OAuth2 Solves

OAuth2 allows users to grant LIMITED access to their resources WITHOUT sharing credentials:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WITH OAUTH2                                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. User clicks "Connect Google Photos" on PrintMyPhotos                │
│                                                                          │
│  2. User is redirected to Google's login page                           │
│     (Password entered on Google, NOT PrintMyPhotos)                     │
│                                                                          │
│  3. Google asks: "PrintMyPhotos wants to:"                              │
│     ✓ View your Google Photos                                           │
│     ✗ NOT access your email                                             │
│     ✗ NOT change your password                                          │
│     [Allow] [Deny]                                                       │
│                                                                          │
│  4. User clicks Allow                                                    │
│                                                                          │
│  5. Google gives PrintMyPhotos a LIMITED access token                   │
│     - Only works for photos                                              │
│     - Expires in 1 hour                                                  │
│     - Can be revoked anytime                                             │
│                                                                          │
│  Benefits:                                                               │
│  ✅ Password never shared with third party                              │
│  ✅ Limited scope (only photos, not email)                              │
│  ✅ Time-limited access                                                  │
│  ✅ Revocable without changing password                                 │
│  ✅ User sees exactly what they're granting                             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### What OpenID Connect Adds

OAuth2 only handles authorization (access to resources). OpenID Connect (OIDC) adds authentication (proving identity):

- **OAuth2**: "This token lets you access photos"
- **OIDC**: "This token proves the user is alice@gmail.com"

This is why "Login with Google" uses OIDC, not just OAuth2.

---

## 2️⃣ Intuition and Mental Model

### The Valet Key Analogy

Think of OAuth2 like a valet key for your car:

**Your Master Key (Password)**:
- Starts the car
- Opens the trunk
- Opens the glove box
- Unlocks all doors
- Full access to everything

**Valet Key (OAuth2 Token)**:
- Starts the car
- Can't open the trunk
- Can't open the glove box
- Limited driving range (some cars)
- You can deactivate it anytime

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OAUTH2 AS VALET KEY                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Resource Owner (You)                                                    │
│       │                                                                  │
│       │ "I want the valet to park my car"                               │
│       │                                                                  │
│       ▼                                                                  │
│  Authorization Server (Car Manufacturer)                                │
│       │                                                                  │
│       │ "Here's a valet key with limited access"                        │
│       │                                                                  │
│       ▼                                                                  │
│  Client (Valet/Third-Party App)                                         │
│       │                                                                  │
│       │ Uses valet key to access                                        │
│       │                                                                  │
│       ▼                                                                  │
│  Resource Server (Your Car/Google Photos)                               │
│       │                                                                  │
│       │ Grants limited access based on key                              │
│       │                                                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ How OAuth2 Works Internally

### OAuth2 Roles

| Role | Description | Example |
|------|-------------|---------|
| **Resource Owner** | The user who owns the data | You |
| **Client** | The app wanting access | PrintMyPhotos.com |
| **Authorization Server** | Issues tokens after user consent | accounts.google.com |
| **Resource Server** | Hosts the protected resources | photos.googleapis.com |

### OAuth2 Grant Types (Flows)

OAuth2 defines several flows for different scenarios:

| Flow | Use Case | Security Level |
|------|----------|----------------|
| **Authorization Code** | Web apps with backend | Highest |
| **Authorization Code + PKCE** | Mobile/SPA apps | High |
| **Client Credentials** | Machine-to-machine | High (no user) |
| **Implicit** | Legacy SPAs | ❌ Deprecated |
| **Resource Owner Password** | Legacy/trusted apps | ❌ Deprecated |

### Authorization Code Flow (Most Common)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    AUTHORIZATION CODE FLOW                                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  User          Client              Auth Server         Resource Server   │
│   │             │                       │                    │           │
│   │ 1. Click    │                       │                    │           │
│   │ "Login"     │                       │                    │           │
│   │────────────>│                       │                    │           │
│   │             │                       │                    │           │
│   │             │ 2. Redirect to        │                    │           │
│   │             │    Auth Server        │                    │           │
│   │<────────────│                       │                    │           │
│   │             │                       │                    │           │
│   │ 3. Login + Consent                  │                    │           │
│   │────────────────────────────────────>│                    │           │
│   │             │                       │                    │           │
│   │ 4. Redirect with Authorization Code │                    │           │
│   │<────────────────────────────────────│                    │           │
│   │             │                       │                    │           │
│   │ 5. Send Code│                       │                    │           │
│   │────────────>│                       │                    │           │
│   │             │                       │                    │           │
│   │             │ 6. Exchange Code      │                    │           │
│   │             │    for Tokens         │                    │           │
│   │             │──────────────────────>│                    │           │
│   │             │                       │                    │           │
│   │             │ 7. Access Token       │                    │           │
│   │             │    + Refresh Token    │                    │           │
│   │             │<──────────────────────│                    │           │
│   │             │                       │                    │           │
│   │             │ 8. API Request with   │                    │           │
│   │             │    Access Token       │                    │           │
│   │             │───────────────────────────────────────────>│           │
│   │             │                       │                    │           │
│   │             │ 9. Protected Resource │                    │           │
│   │             │<───────────────────────────────────────────│           │
│   │             │                       │                    │           │
│   │ 10. Data    │                       │                    │           │
│   │<────────────│                       │                    │           │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Step-by-Step Breakdown**:

**Step 2: Authorization Request**
```
GET https://accounts.google.com/o/oauth2/v2/auth?
  response_type=code                           # We want an authorization code
  &client_id=YOUR_CLIENT_ID                    # Identifies your app
  &redirect_uri=https://yourapp.com/callback   # Where to send the code
  &scope=https://www.googleapis.com/auth/photoslibrary.readonly  # What access
  &state=xyz123                                # CSRF protection
  &access_type=offline                         # Request refresh token
```

**Step 4: Authorization Response (Redirect)**
```
HTTP/1.1 302 Found
Location: https://yourapp.com/callback?
  code=4/P7q7W91a-oMsCeLvIaQm6bTrgtp7    # Authorization code (short-lived)
  &state=xyz123                           # Same state for CSRF verification
```

**Step 6: Token Exchange**
```http
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=4/P7q7W91a-oMsCeLvIaQm6bTrgtp7
&redirect_uri=https://yourapp.com/callback
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
```

**Step 7: Token Response**
```json
{
  "access_token": "ya29.a0AfH6SMBx...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "1//0eVj...",
  "scope": "https://www.googleapis.com/auth/photoslibrary.readonly"
}
```

### PKCE (Proof Key for Code Exchange)

PKCE protects against authorization code interception attacks, essential for mobile/SPA apps:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PKCE FLOW                                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Client generates random "code_verifier" (43-128 chars)              │
│     code_verifier = "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"       │
│                                                                          │
│  2. Client creates "code_challenge" from verifier                       │
│     code_challenge = BASE64URL(SHA256(code_verifier))                   │
│     code_challenge = "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"      │
│                                                                          │
│  3. Authorization request includes code_challenge                       │
│     GET /authorize?                                                      │
│       code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM        │
│       &code_challenge_method=S256                                        │
│       &...                                                               │
│                                                                          │
│  4. Token exchange includes code_verifier                               │
│     POST /token                                                          │
│       code=...                                                           │
│       &code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk        │
│                                                                          │
│  5. Auth server verifies:                                                │
│     BASE64URL(SHA256(code_verifier)) == stored code_challenge           │
│                                                                          │
│  Why this works:                                                         │
│  • Attacker intercepts authorization code                               │
│  • Attacker doesn't have code_verifier (never transmitted)              │
│  • Attacker can't exchange code for tokens                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Client Credentials Flow (Machine-to-Machine)

For service-to-service communication where no user is involved:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CLIENT CREDENTIALS FLOW                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Service A                    Auth Server                   Service B   │
│      │                            │                             │       │
│      │ 1. POST /token             │                             │       │
│      │    client_id=...           │                             │       │
│      │    client_secret=...       │                             │       │
│      │    grant_type=             │                             │       │
│      │      client_credentials    │                             │       │
│      │    scope=service-b:read    │                             │       │
│      │───────────────────────────>│                             │       │
│      │                            │                             │       │
│      │ 2. Access Token            │                             │       │
│      │<───────────────────────────│                             │       │
│      │                            │                             │       │
│      │ 3. API Request with Token  │                             │       │
│      │─────────────────────────────────────────────────────────>│       │
│      │                            │                             │       │
│      │ 4. Response                │                             │       │
│      │<─────────────────────────────────────────────────────────│       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ OpenID Connect (OIDC)

### What OIDC Adds to OAuth2

OIDC is an identity layer on top of OAuth2:

| OAuth2 | OpenID Connect |
|--------|----------------|
| Access Token (authorization) | ID Token (authentication) |
| "What can you access?" | "Who are you?" |
| Opaque or JWT | Always JWT |
| No standard user info | Standard claims (name, email, etc.) |

### ID Token Structure

```json
{
  "iss": "https://accounts.google.com",      // Issuer
  "sub": "110169484474386276334",            // Subject (unique user ID)
  "aud": "YOUR_CLIENT_ID",                   // Audience (your app)
  "exp": 1516242622,                         // Expiration
  "iat": 1516239022,                         // Issued at
  "auth_time": 1516239022,                   // When user authenticated
  "nonce": "abc123",                         // Replay protection
  
  // Standard OIDC claims
  "email": "alice@gmail.com",
  "email_verified": true,
  "name": "Alice Smith",
  "picture": "https://lh3.googleusercontent.com/...",
  "given_name": "Alice",
  "family_name": "Smith",
  "locale": "en"
}
```

### OIDC Scopes

| Scope | Claims Returned |
|-------|-----------------|
| `openid` | sub (required for OIDC) |
| `profile` | name, family_name, given_name, picture, etc. |
| `email` | email, email_verified |
| `address` | address |
| `phone` | phone_number, phone_number_verified |

### OIDC Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/.well-known/openid-configuration` | Discovery document |
| `/authorize` | Start authentication |
| `/token` | Exchange code for tokens |
| `/userinfo` | Get user profile |
| `/jwks` | Public keys for token verification |

---

## 5️⃣ Simulation-First Explanation

### Complete "Login with Google" Flow

Let's trace exactly what happens when a user clicks "Login with Google":

**Step 1: User Clicks Login Button**
```html
<a href="https://accounts.google.com/o/oauth2/v2/auth?
  client_id=123456789.apps.googleusercontent.com
  &redirect_uri=https://myapp.com/callback
  &response_type=code
  &scope=openid%20email%20profile
  &state=random_csrf_token
  &nonce=random_replay_protection">
  Login with Google
</a>
```

**Step 2: User Sees Google Consent Screen**
```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Google Sign In                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Choose an account to continue to MyApp                                 │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  👤 alice@gmail.com                                              │    │
│  │     Alice Smith                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  MyApp wants to:                                                         │
│  ✓ See your email address                                               │
│  ✓ See your personal info, including any you've made public            │
│                                                                          │
│  [Cancel]                                            [Allow]             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Step 3: Google Redirects Back with Code**
```
HTTP/1.1 302 Found
Location: https://myapp.com/callback?
  code=4/0AX4XfWj...
  &state=random_csrf_token
  &scope=email%20profile%20openid
```

**Step 4: Backend Exchanges Code for Tokens**
```http
POST https://oauth2.googleapis.com/token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

code=4/0AX4XfWj...
&client_id=123456789.apps.googleusercontent.com
&client_secret=GOCSPX-...
&redirect_uri=https://myapp.com/callback
&grant_type=authorization_code
```

**Step 5: Google Returns Tokens**
```json
{
  "access_token": "ya29.a0AfH6SMBx...",
  "expires_in": 3599,
  "token_type": "Bearer",
  "scope": "openid email profile",
  "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJzdWIiOiIxMTAxNjk0ODQ0NzQzODYyNzYzMzQiLCJhdWQiOiIxMjM0NTY3ODkuYXBwcy5nb29nbGV1c2VyY29udGVudC5jb20iLCJlbWFpbCI6ImFsaWNlQGdtYWlsLmNvbSIsImVtYWlsX3ZlcmlmaWVkIjp0cnVlLCJuYW1lIjoiQWxpY2UgU21pdGgiLCJwaWN0dXJlIjoiaHR0cHM6Ly9saDMuZ29vZ2xldXNlcmNvbnRlbnQuY29tLy4uLiIsImlhdCI6MTUxNjIzOTAyMiwiZXhwIjoxNTE2MjQyNjIyfQ.signature"
}
```

**Step 6: Backend Validates ID Token**
```java
// Decode and validate ID token
DecodedJWT idToken = JWT.decode(response.getIdToken());

// Verify signature using Google's public keys (from JWKS endpoint)
// Verify issuer is https://accounts.google.com
// Verify audience is our client_id
// Verify not expired
// Verify nonce matches what we sent

String email = idToken.getClaim("email").asString();
String name = idToken.getClaim("name").asString();
String googleUserId = idToken.getSubject();

// Create or find user in our database
User user = userService.findOrCreateByGoogleId(googleUserId, email, name);

// Create our own session/JWT
String ourToken = jwtService.createToken(user);
```

---

## 6️⃣ How to Implement OAuth2/OIDC in Java

### Spring Security OAuth2 Client

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope:
              - openid
              - email
              - profile
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope:
              - user:email
              - read:user
        provider:
          # Google is auto-configured, but you can customize
          google:
            authorization-uri: https://accounts.google.com/o/oauth2/v2/auth
            token-uri: https://oauth2.googleapis.com/token
            user-info-uri: https://openidconnect.googleapis.com/v1/userinfo
            jwk-set-uri: https://www.googleapis.com/oauth2/v3/certs
```

### Security Configuration

```java
package com.example.security.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class OAuth2SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard", true)
                .failureUrl("/login?error=true")
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService())
                )
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/")
                .invalidateHttpSession(true)
                .clearAuthentication(true)
            );
        
        return http.build();
    }
    
    @Bean
    public CustomOAuth2UserService customOAuth2UserService() {
        return new CustomOAuth2UserService();
    }
}
```

### Custom OAuth2 User Service

```java
package com.example.security.service;

import org.springframework.security.oauth2.client.userinfo.DefaultOAuth2UserService;
import org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest;
import org.springframework.security.oauth2.core.OAuth2AuthenticationException;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Service;

import java.util.Map;

/**
 * Custom service to process OAuth2 user info and create/update local user.
 */
@Service
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    private final UserRepository userRepository;

    public CustomOAuth2UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) 
            throws OAuth2AuthenticationException {
        
        // Load user info from OAuth2 provider
        OAuth2User oauth2User = super.loadUser(userRequest);
        
        // Get provider name (google, github, etc.)
        String provider = userRequest.getClientRegistration().getRegistrationId();
        
        // Extract user attributes based on provider
        Map<String, Object> attributes = oauth2User.getAttributes();
        
        String providerId;
        String email;
        String name;
        String picture;
        
        switch (provider) {
            case "google" -> {
                providerId = (String) attributes.get("sub");
                email = (String) attributes.get("email");
                name = (String) attributes.get("name");
                picture = (String) attributes.get("picture");
            }
            case "github" -> {
                providerId = String.valueOf(attributes.get("id"));
                email = (String) attributes.get("email");
                name = (String) attributes.get("name");
                picture = (String) attributes.get("avatar_url");
            }
            default -> throw new OAuth2AuthenticationException("Unknown provider: " + provider);
        }
        
        // Find or create user in our database
        User user = userRepository.findByProviderAndProviderId(provider, providerId)
            .orElseGet(() -> {
                User newUser = new User();
                newUser.setProvider(provider);
                newUser.setProviderId(providerId);
                newUser.setEmail(email);
                newUser.setName(name);
                newUser.setPicture(picture);
                return userRepository.save(newUser);
            });
        
        // Update user info if changed
        user.setName(name);
        user.setPicture(picture);
        userRepository.save(user);
        
        // Return custom principal with our user data
        return new CustomOAuth2User(oauth2User, user);
    }
}
```

### Implementing OAuth2 Resource Server

For APIs that accept OAuth2 tokens:

```yaml
# application.yml for Resource Server
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://accounts.google.com
          # OR specify JWKS directly
          jwk-set-uri: https://www.googleapis.com/oauth2/v3/certs
```

```java
package com.example.api.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            );
        
        return http.build();
    }
    
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = 
            new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");
        
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        return converter;
    }
}
```

### Building Your Own OAuth2 Authorization Server

```java
package com.example.authserver.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.oauth2.core.AuthorizationGrantType;
import org.springframework.security.oauth2.core.ClientAuthenticationMethod;
import org.springframework.security.oauth2.server.authorization.client.InMemoryRegisteredClientRepository;
import org.springframework.security.oauth2.server.authorization.client.RegisteredClient;
import org.springframework.security.oauth2.server.authorization.client.RegisteredClientRepository;
import org.springframework.security.oauth2.server.authorization.config.annotation.web.configuration.OAuth2AuthorizationServerConfiguration;
import org.springframework.security.oauth2.server.authorization.settings.AuthorizationServerSettings;
import org.springframework.security.oauth2.server.authorization.settings.TokenSettings;
import org.springframework.security.web.SecurityFilterChain;

import java.time.Duration;
import java.util.UUID;

/**
 * OAuth2 Authorization Server configuration.
 * 
 * This creates an authorization server that can:
 * - Register clients (applications)
 * - Authenticate users
 * - Issue access tokens and refresh tokens
 * - Support authorization code flow with PKCE
 */
@Configuration
public class AuthorizationServerConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http) 
            throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);
        return http.formLogin(Customizer.withDefaults()).build();
    }

    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        // In production, store clients in database
        RegisteredClient webClient = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("web-client")
            .clientSecret("{bcrypt}$2a$10$...")  // Encoded secret
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .redirectUri("https://myapp.com/callback")
            .scope("openid")
            .scope("profile")
            .scope("email")
            .tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofMinutes(15))
                .refreshTokenTimeToLive(Duration.ofDays(7))
                .reuseRefreshTokens(false)  // Rotation
                .build())
            .build();
        
        RegisteredClient mobileClient = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("mobile-client")
            // No client secret for public clients (mobile/SPA)
            .clientAuthenticationMethod(ClientAuthenticationMethod.NONE)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .redirectUri("myapp://callback")
            .scope("openid")
            .scope("profile")
            // Require PKCE for public clients
            .clientSettings(ClientSettings.builder()
                .requireProofKey(true)
                .build())
            .build();
        
        RegisteredClient serviceClient = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("service-client")
            .clientSecret("{bcrypt}$2a$10$...")
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
            .scope("service:read")
            .scope("service:write")
            .build();
        
        return new InMemoryRegisteredClientRepository(webClient, mobileClient, serviceClient);
    }

    @Bean
    public AuthorizationServerSettings authorizationServerSettings() {
        return AuthorizationServerSettings.builder()
            .issuer("https://auth.example.com")
            .build();
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common OAuth2 Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not validating state | CSRF attacks | Always verify state matches |
| Not using PKCE | Code interception | Always use PKCE for public clients |
| Storing tokens in localStorage | XSS token theft | Use HttpOnly cookies or memory |
| Long-lived access tokens | Extended compromise | Short expiration + refresh tokens |
| Not validating redirect_uri | Open redirect attacks | Exact match, no wildcards |
| Implicit flow for SPAs | Token exposure in URL | Use auth code + PKCE |
| Not validating ID token | Identity spoofing | Verify signature, issuer, audience |

### Security Considerations

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OAUTH2 SECURITY CHECKLIST                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ✓ Use HTTPS for ALL OAuth2 communication                               │
│  ✓ Validate state parameter to prevent CSRF                             │
│  ✓ Use PKCE for mobile and SPA applications                            │
│  ✓ Validate redirect_uri exactly (no wildcards)                         │
│  ✓ Keep client_secret secure (never in frontend code)                   │
│  ✓ Use short-lived access tokens (< 1 hour)                             │
│  ✓ Rotate refresh tokens on each use                                    │
│  ✓ Validate ID token signature, issuer, audience, expiration           │
│  ✓ Use nonce to prevent replay attacks                                  │
│  ✓ Implement token revocation for logout                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8️⃣ When NOT to Use OAuth2

### Cases Where OAuth2 is Overkill

1. **Internal services only**: If you control all services, mTLS or API keys may be simpler
2. **Simple username/password**: If you don't need third-party login, JWT auth is simpler
3. **Machine-to-machine with fixed peers**: API keys or mTLS may suffice

### When to Use What

| Scenario | Solution |
|----------|----------|
| "Login with Google/Facebook" | OIDC |
| Third-party API access | OAuth2 Authorization Code |
| Mobile app auth | OAuth2 + PKCE |
| SPA auth | OAuth2 + PKCE (BFF pattern recommended) |
| Service-to-service | OAuth2 Client Credentials or mTLS |
| Internal users only | SAML or direct JWT |

---

## 9️⃣ Comparison with Alternatives

### OAuth2 vs SAML

| Aspect | OAuth2/OIDC | SAML |
|--------|-------------|------|
| Format | JSON/JWT | XML |
| Transport | HTTP redirects | HTTP POST/Redirect |
| Mobile friendly | Yes | No |
| Complexity | Moderate | High |
| Enterprise SSO | Growing | Established |
| Use case | Consumer apps, APIs | Enterprise SSO |

### OAuth2 vs API Keys

| Aspect | OAuth2 | API Keys |
|--------|--------|----------|
| User context | Yes | No |
| Fine-grained scopes | Yes | Limited |
| Expiration | Built-in | Manual |
| Revocation | Standard | Custom |
| Complexity | Higher | Lower |

---

## 🔟 Interview Follow-Up Questions

### L4 (Entry Level) Questions

**Q: What's the difference between OAuth2 and OpenID Connect?**
A: OAuth2 is an authorization framework that lets users grant limited access to their resources without sharing passwords. It answers "What can this app access?" OpenID Connect (OIDC) is an identity layer built on top of OAuth2 that adds authentication. It answers "Who is this user?" OIDC adds the ID token (a JWT containing user identity claims) and standardized endpoints for user info.

**Q: Why do we need the authorization code flow? Why not just return the token directly?**
A: The authorization code flow adds security by keeping tokens off the browser URL. If we returned tokens directly (implicit flow), they'd appear in the URL fragment, browser history, and server logs. The code flow returns a short-lived authorization code that's exchanged for tokens via a secure back-channel request that includes the client secret.

### L5 (Mid Level) Questions

**Q: Explain PKCE and why it's important for mobile apps.**
A: PKCE (Proof Key for Code Exchange) protects against authorization code interception attacks. Mobile apps can't securely store a client_secret because the app binary can be decompiled. With PKCE, the client generates a random code_verifier, sends a hash of it (code_challenge) with the authorization request, and sends the original verifier when exchanging the code. An attacker who intercepts the code can't exchange it without the verifier. This is now recommended for ALL public clients, including SPAs.

**Q: How would you implement secure logout in an OAuth2 system?**
A: Secure logout requires multiple steps:
1. Clear local session/tokens in the client
2. Revoke refresh token at the authorization server (token revocation endpoint)
3. Optionally redirect to authorization server's logout endpoint (OIDC has standard endpoint)
4. Clear any server-side session
5. For SSO, consider front-channel or back-channel logout to notify all clients

### L6 (Senior) Questions

**Q: Design an OAuth2-based authentication system for a microservices architecture.**
A:
1. **Authorization Server**: Centralized (Keycloak, Auth0, or custom)
2. **Token format**: JWT for stateless validation
3. **Token flow**: 
   - User-facing: Authorization Code + PKCE
   - Service-to-service: Client Credentials
4. **Token validation**: Each service validates JWT signature using JWKS
5. **Token propagation**: Pass access token in requests between services
6. **Scopes**: Fine-grained scopes per API/resource
7. **Token exchange**: For service chains needing different scopes
8. **Monitoring**: Track token issuance, validation failures, suspicious patterns
9. **Revocation**: Short-lived tokens + refresh token rotation + blacklist for emergencies

**Q: What are the security considerations when implementing OAuth2 for a single-page application?**
A:
1. **No client secret**: SPAs are public clients, can't keep secrets
2. **PKCE required**: Protects authorization code
3. **Token storage**: Memory only for access tokens, refresh via HttpOnly cookie
4. **BFF pattern**: Consider Backend-for-Frontend to handle tokens server-side
5. **Short-lived tokens**: 5-15 minutes for access tokens
6. **Silent refresh**: Use hidden iframe or refresh token for seamless renewal
7. **CSP headers**: Prevent XSS that could steal tokens
8. **Redirect URI validation**: Exact match, no wildcards

---

## 1️⃣1️⃣ One Clean Mental Summary

OAuth2 solves the problem of granting limited access to your resources without sharing your password. Instead of giving a third-party app your Google password, you authorize Google to give them a limited, time-bound, revocable token. OpenID Connect adds identity on top, enabling "Login with Google" by proving who you are via an ID token. For web apps, use Authorization Code flow. For mobile/SPAs, add PKCE. For service-to-service, use Client Credentials. Always validate tokens properly: check signature, issuer, audience, expiration, and scopes.

---

## Quick Reference

```
OAuth2 Endpoints:
- /authorize     - Start authorization (user consent)
- /token         - Exchange code for tokens
- /revoke        - Revoke tokens
- /introspect    - Check token validity

OIDC Endpoints:
- /.well-known/openid-configuration - Discovery
- /userinfo      - Get user profile
- /jwks          - Public keys

Common Scopes:
- openid         - Required for OIDC
- profile        - Name, picture, etc.
- email          - Email address
- offline_access - Request refresh token
```

---

## References

- [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- [PKCE RFC 7636](https://tools.ietf.org/html/rfc7636)
- [OAuth 2.0 Security Best Practices](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
- [Spring Authorization Server](https://spring.io/projects/spring-authorization-server)

