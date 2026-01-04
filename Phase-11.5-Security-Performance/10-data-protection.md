# 🔐 Data Protection: Encryption, PII Handling, and Privacy Compliance

---

## 0️⃣ Prerequisites

Before diving into data protection, you should understand:

- **Encryption**: The process of converting readable data (plaintext) into unreadable data (ciphertext) that can only be decrypted with a key. We'll explain symmetric vs asymmetric encryption.
- **Secrets Management**: How to securely store and retrieve encryption keys. Covered in `09-secrets-management.md`.
- **HTTPS/TLS**: How data is encrypted in transit. Covered in `01-https-tls.md`.
- **Database Basics**: Understanding of relational databases and how data is stored. Covered in Phase 3.

**Quick Refresher**: Data protection ensures sensitive information (passwords, credit cards, personal data) is encrypted when stored (at rest) and when transmitted (in transit), and that access is controlled and audited.

---

## 1️⃣ What Problem Does Data Protection Exist to Solve?

### The Core Problem: Data Breaches Are Catastrophic

Every day, applications handle sensitive data:
- **Personal Identifiable Information (PII)**: Names, emails, addresses, SSNs
- **Financial Data**: Credit card numbers, bank accounts
- **Health Information**: Medical records (HIPAA protected)
- **Authentication Credentials**: Passwords, tokens
- **Business Secrets**: Trade secrets, customer lists

### What Systems Looked Like Before Data Protection

**The Dark Ages (Unencrypted Data)**:

```sql
-- ❌ TERRIBLE: Plain text storage
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(255),
    password VARCHAR(255),  -- Stored as plain text!
    credit_card VARCHAR(16),  -- Unencrypted!
    ssn VARCHAR(9)  -- No protection!
);

INSERT INTO users VALUES (
    1, 
    'alice@example.com',
    'password123',  -- Anyone with DB access can see this
    '4532123456789012',  -- Credit card in plain text
    '123456789'  -- SSN visible
);
```

**Problems with this approach**:
1. Database breach = all data exposed immediately
2. Backup theft = all data readable
3. Database admin can see everything
4. No way to detect unauthorized access
5. Compliance violations (GDPR, HIPAA, PCI-DSS)

### Real-World Data Breach Examples

**2013 Adobe Breach**: 153 million user records exposed
- Passwords encrypted with weak algorithm (reversible)
- Attackers decrypted 38 million passwords
- Impact: Users' passwords compromised, reused passwords exposed

**2017 Equifax Breach**: 147 million people affected
- SSNs, birth dates, addresses stored unencrypted
- Attackers had access for 76 days before detection
- Impact: Identity theft, financial fraud, $700M settlement

**2018 Facebook/Cambridge Analytica**: 87 million profiles exposed
- Personal data (likes, friends, location) accessed without consent
- No encryption on shared data
- Impact: Privacy violations, GDPR fines, public trust loss

### What Breaks Without Data Protection

| Attack Scenario | Impact | Without Protection | With Protection |
|----------------|--------|-------------------|-----------------|
| **Database Breach** | Attacker steals database | All data readable | Encrypted data useless without keys |
| **Backup Theft** | Attacker steals backup | All data readable | Encrypted backup requires keys |
| **Insider Threat** | Employee accesses data | Can read everything | Access logged, encrypted data |
| **Network Sniffing** | Attacker intercepts traffic | Can read all data | TLS encrypts in transit |
| **Compliance Audit** | Regulator reviews system | Fails audit, fines | Passes audit, compliant |

### The Three Pillars of Data Protection

1. **Confidentiality**: Data is encrypted and only accessible to authorized parties
2. **Integrity**: Data cannot be modified without detection
3. **Availability**: Authorized users can access data when needed

---

## 2️⃣ Intuition and Mental Model

### The Safe Deposit Box Analogy

Think of data protection like a bank's safe deposit box system:

**Without Protection (Plain Text)**:
- Your valuables are in a glass box
- Anyone walking by can see everything
- No record of who looked
- If the box is stolen, everything is lost

**With Protection (Encrypted)**:
1. **Encryption at Rest**: Valuables locked in a safe (encrypted storage)
2. **Access Control**: Only authorized people get keys (IAM, authentication)
3. **Audit Trail**: Every access is logged (who, when, what)
4. **Encryption in Transit**: Valuables transported in armored vehicles (TLS)
5. **Backup Protection**: Backup copies also locked in safes (encrypted backups)

### The Envelope Analogy for Encryption

**Plain Text (No Encryption)**:
```
Letter: "Your password is secret123"
Status: Anyone can read it
```

**Encrypted (With Protection)**:
```
Letter: "aGVsbG8gd29ybGQK" (encrypted)
Key: Only authorized person has the key
Status: Without key, it's gibberish
```

Even if someone steals the letter, without the key, they can't read it.

---

## 3️⃣ How Data Protection Works Internally

### Encryption at Rest

**Step 1: Data Encryption**
```
Plaintext: "password123"
         ↓
Encryption Key: (stored in KMS)
         ↓
Encryption Algorithm: AES-256
         ↓
Ciphertext: "aGVsbG8gd29ybGQK"
         ↓
Storage: Encrypted value in database
```

**Step 2: Key Management**
- Encryption keys stored in Key Management Service (KMS)
- Keys never leave the KMS (Hardware Security Module)
- Keys are rotated periodically
- Access to keys is logged and audited

**Step 3: Data Access**
```
Application Request
         ↓
Authentication (verify identity)
         ↓
Authorization (check permissions)
         ↓
Retrieve Encrypted Data
         ↓
Request Decryption Key from KMS
         ↓
Decrypt Data
         ↓
Return Plaintext to Application
         ↓
Audit Log (who accessed what)
```

### Encryption in Transit

**TLS/HTTPS Flow**:
```
Client                    Server
   │                         │
   │─── TLS Handshake ───────▶│
   │                         │
   │◀── Certificate ─────────│
   │                         │
   │─── Encrypted Request ──▶│
   │                         │
   │◀── Encrypted Response ──│
```

- All data encrypted before transmission
- Only client and server can decrypt
- Protects against man-in-the-middle attacks

### Data Masking

**Purpose**: Show partial data for testing/debugging without exposing full values

**Example**:
```
Original: "4532123456789012" (credit card)
Masked:   "4532********9012" (last 4 digits visible)
```

**Implementation**:
```java
public String maskCreditCard(String cardNumber) {
    if (cardNumber == null || cardNumber.length() < 4) {
        return "****";
    }
    String last4 = cardNumber.substring(cardNumber.length() - 4);
    return "****" + last4;
}
```

---

## 4️⃣ Simulation-First Explanation

### Scenario: User Registration with PII

Let's trace a user registration that handles sensitive data:

**Step 1: User Submits Registration Form**
```html
<form action="/register" method="POST">
    <input name="email" value="alice@example.com">
    <input name="password" value="secret123" type="password">
    <input name="ssn" value="123456789">
    <input name="creditCard" value="4532123456789012">
</form>
```

**Step 2: Data Transmitted Over HTTPS (Encryption in Transit)**
```
Browser                    Server
   │                         │
   │─── HTTPS Request ──────▶│
   │  (TLS encrypted)        │
   │  POST /register         │
   │  email=alice@...        │
   │  password=secret123      │
   │  ssn=123456789          │
   │  creditCard=4532...     │
   │                         │
   │  [All data encrypted    │
   │   during transmission]   │
```

**Step 3: Server Receives and Validates Data**
```java
@PostMapping("/register")
public ResponseEntity<User> register(@RequestBody RegisterRequest request) {
    // 1. Validate input
    validateEmail(request.getEmail());
    validateSSN(request.getSsn());
    validateCreditCard(request.getCreditCard());
    
    // 2. Hash password (one-way encryption)
    String hashedPassword = passwordEncoder.encode(request.getPassword());
    // Result: "$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy"
    
    // 3. Encrypt sensitive data
    String encryptedSSN = encryptionService.encrypt(request.getSsn());
    // Result: "aGVsbG8gd29ybGQK" (encrypted)
    
    String encryptedCreditCard = encryptionService.encrypt(request.getCreditCard());
    // Result: "bXlzZWNyZXRkYXRhCg==" (encrypted)
    
    // 4. Store in database
    User user = new User();
    user.setEmail(request.getEmail());
    user.setPassword(hashedPassword);  // Hashed, not encrypted
    user.setSsn(encryptedSSN);  // Encrypted
    user.setCreditCard(encryptedCreditCard);  // Encrypted
    
    userRepository.save(user);
    
    return ResponseEntity.ok(user);
}
```

**Step 4: Data Stored in Database (Encryption at Rest)**
```sql
-- What's actually stored
INSERT INTO users (email, password, ssn, credit_card) VALUES (
    'alice@example.com',
    '$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy',  -- Hashed
    'aGVsbG8gd29ybGQK',  -- Encrypted SSN
    'bXlzZWNyZXRkYXRhCg=='  -- Encrypted credit card
);
```

**Step 5: Data Retrieved and Decrypted**
```java
@GetMapping("/users/{id}")
public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
    // 1. Authenticate and authorize
    User currentUser = getCurrentUser();
    if (!currentUser.isAdmin() && !currentUser.getId().equals(id)) {
        throw new ForbiddenException();
    }
    
    // 2. Retrieve encrypted data
    User user = userRepository.findById(id).orElseThrow();
    
    // 3. Decrypt sensitive fields
    String decryptedSSN = encryptionService.decrypt(user.getSsn());
    String decryptedCreditCard = encryptionService.decrypt(user.getCreditCard());
    
    // 4. Mask data for response (don't send full values)
    UserResponse response = new UserResponse();
    response.setEmail(user.getEmail());
    response.setSsn(maskSSN(decryptedSSN));  // "***-**-6789"
    response.setCreditCard(maskCreditCard(decryptedCreditCard));  // "****9012"
    
    // 5. Audit log
    auditLog.log("USER_ACCESSED", currentUser.getId(), id);
    
    return ResponseEntity.ok(response);
}
```

### What Happens in a Breach Scenario

**Without Encryption**:
```
Attacker steals database
         ↓
Opens database file
         ↓
Sees all data in plain text:
  - email: alice@example.com
  - password: secret123
  - ssn: 123456789
  - creditCard: 4532123456789012
         ↓
All data compromised immediately
```

**With Encryption**:
```
Attacker steals database
         ↓
Opens database file
         ↓
Sees encrypted data:
  - email: alice@example.com (not sensitive, plain text OK)
  - password: $2a$10$N9qo... (hashed, can't reverse)
  - ssn: aGVsbG8gd29ybGQK (encrypted, needs key)
  - creditCard: bXlzZWNyZXRkYXRhCg== (encrypted, needs key)
         ↓
Tries to decrypt
         ↓
Needs encryption key from KMS
         ↓
KMS requires authentication
         ↓
Attacker can't access key
         ↓
Data is useless without key
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Real-World Implementations

**Stripe (Payment Processing)**:
- All credit card numbers encrypted with AES-256
- Keys stored in Hardware Security Modules (HSM)
- PCI-DSS Level 1 compliant
- Data never decrypted unless necessary
- Reference: Stripe Security Documentation

**Healthcare Systems (HIPAA)**:
- Patient data encrypted at rest and in transit
- Access controls: only authorized medical staff
- Audit logs: every access to patient records
- Data retention policies: delete after required period
- Example: Epic Systems, Cerner

**Banking Systems**:
- Multi-layer encryption (database, application, network)
- Key rotation every 90 days
- Separate encryption for different data types
- Compliance: SOX, PCI-DSS, GLBA

### Common Patterns

**Pattern 1: Field-Level Encryption**
```java
@Entity
public class User {
    @Id
    private Long id;
    
    private String email;  // Not encrypted (not sensitive)
    
    @Encrypted  // Custom annotation
    private String ssn;  // Encrypted
    
    @Encrypted
    private String creditCard;  // Encrypted
    
    @Hashed  // One-way hash
    private String password;  // Hashed, not encrypted
}
```

**Pattern 2: Application-Level Encryption**
```java
@Service
public class EncryptionService {
    private final KeyManagementService kms;
    
    public String encrypt(String plaintext) {
        // 1. Get encryption key from KMS
        SecretKey key = kms.getKey("data-encryption-key");
        
        // 2. Encrypt data
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.ENCRYPT_MODE, key);
        byte[] encrypted = cipher.doFinal(plaintext.getBytes());
        
        // 3. Return base64 encoded
        return Base64.getEncoder().encodeToString(encrypted);
    }
    
    public String decrypt(String ciphertext) {
        // Reverse process
        SecretKey key = kms.getKey("data-encryption-key");
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.DECRYPT_MODE, key);
        byte[] decrypted = cipher.doFinal(Base64.getDecoder().decode(ciphertext));
        return new String(decrypted);
    }
}
```

**Pattern 3: Database-Level Encryption (Transparent Data Encryption)**
```sql
-- PostgreSQL with pgcrypto
CREATE EXTENSION pgcrypto;

-- Encrypt on insert
INSERT INTO users (ssn) 
VALUES (pgp_sym_encrypt('123456789', 'encryption_key'));

-- Decrypt on select
SELECT pgp_sym_decrypt(ssn, 'encryption_key') FROM users;
```

**Pattern 4: PII Detection and Masking**
```java
@Service
public class PIIDetectionService {
    
    public String detectAndMask(String text) {
        // Detect SSN pattern: XXX-XX-XXXX
        text = text.replaceAll("\\b\\d{3}-\\d{2}-\\d{4}\\b", "***-**-****");
        
        // Detect credit card: XXXX XXXX XXXX XXXX
        text = text.replaceAll("\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b", 
            "**** **** **** $1");
        
        // Detect email (keep domain, mask username)
        text = text.replaceAll("\\b([a-zA-Z0-9._%+-]+)@([a-zA-Z0-9.-]+\\.[a-zA-Z]{2,})\\b",
            "***@$2");
        
        return text;
    }
}
```

---

## 6️⃣ How to Implement or Apply It

### Option 1: Spring Boot with Jasypt (Java Encryption Library)

**Maven Dependencies**:
```xml
<dependencies>
    <dependency>
        <groupId>com.github.ulisesbocchio</groupId>
        <artifactId>jasypt-spring-boot-starter</artifactId>
        <version>3.0.5</version>
    </dependency>
</dependencies>
```

**Java Implementation**:
```java
package com.example.encryption;

import org.jasypt.encryption.StringEncryptor;
import org.jasypt.encryption.pbe.PooledPBEStringEncryptor;
import org.jasypt.encryption.pbe.config.SimpleStringPBEConfig;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class EncryptionConfig {
    
    @Bean("jasyptStringEncryptor")
    public StringEncryptor stringEncryptor() {
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();
        
        // Get encryption password from environment variable
        config.setPassword(System.getenv("ENCRYPTION_PASSWORD"));
        config.setAlgorithm("PBEWithMD5AndDES");
        config.setKeyObtentionIterations("1000");
        config.setPoolSize("1");
        config.setProviderName("SunJCE");
        config.setSaltGeneratorClassName("org.jasypt.salt.RandomSaltGenerator");
        config.setStringOutputType("base64");
        
        encryptor.setConfig(config);
        return encryptor;
    }
}
```

**Service Layer**:
```java
package com.example.service;

import org.jasypt.encryption.StringEncryptor;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

@Service
public class EncryptionService {
    
    private final StringEncryptor encryptor;
    
    public EncryptionService(@Qualifier("jasyptStringEncryptor") StringEncryptor encryptor) {
        this.encryptor = encryptor;
    }
    
    public String encrypt(String plaintext) {
        if (plaintext == null) {
            return null;
        }
        return encryptor.encrypt(plaintext);
    }
    
    public String decrypt(String ciphertext) {
        if (ciphertext == null) {
            return null;
        }
        return encryptor.decrypt(ciphertext);
    }
}
```

**Entity with Encrypted Fields**:
```java
package com.example.entity;

import com.example.service.EncryptionService;
import jakarta.persistence.*;

@Entity
@Table(name = "users")
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String email;
    
    @Column(name = "ssn")
    private String ssn;  // Will be encrypted before saving
    
    @Column(name = "credit_card")
    private String creditCard;  // Will be encrypted
    
    // Getters and setters with encryption
    public String getSsn() {
        return ssn;  // Return encrypted value
    }
    
    public void setSsn(String ssn, EncryptionService encryptionService) {
        this.ssn = encryptionService.encrypt(ssn);
    }
    
    public String getDecryptedSsn(EncryptionService encryptionService) {
        return encryptionService.decrypt(this.ssn);
    }
    
    // Similar for creditCard
    // ... getters and setters
}
```

**application.yml**:
```yaml
jasypt:
  encryptor:
    password: ${ENCRYPTION_PASSWORD}  # From environment variable
    algorithm: PBEWithMD5AndDES

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: app_user
    password: ENC(encrypted_password_here)  # Encrypted in config
```

### Option 2: AWS KMS for Encryption

**Maven Dependencies**:
```xml
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>kms</artifactId>
    <version>2.20.0</version>
</dependency>
```

**Java Implementation**:
```java
package com.example.encryption;

import software.amazon.awssdk.core.SdkBytes;
import software.amazon.awssdk.services.kms.KmsClient;
import software.amazon.awssdk.services.kms.model.EncryptRequest;
import software.amazon.awssdk.services.kms.model.DecryptRequest;
import software.amazon.awssdk.services.kms.model.KmsException;

import java.util.Base64;

public class AWSKMSEncryptionService {
    
    private final KmsClient kmsClient;
    private final String keyId;
    
    public AWSKMSEncryptionService(KmsClient kmsClient, String keyId) {
        this.kmsClient = kmsClient;
        this.keyId = keyId;
    }
    
    public String encrypt(String plaintext) {
        try {
            EncryptRequest request = EncryptRequest.builder()
                .keyId(keyId)
                .plaintext(SdkBytes.fromByteArray(plaintext.getBytes()))
                .build();
            
            var response = kmsClient.encrypt(request);
            byte[] encrypted = response.ciphertextBlob().asByteArray();
            
            return Base64.getEncoder().encodeToString(encrypted);
            
        } catch (KmsException e) {
            throw new RuntimeException("Encryption failed", e);
        }
    }
    
    public String decrypt(String ciphertext) {
        try {
            byte[] encryptedBytes = Base64.getDecoder().decode(ciphertext);
            
            DecryptRequest request = DecryptRequest.builder()
                .ciphertextBlob(SdkBytes.fromByteArray(encryptedBytes))
                .build();
            
            var response = kmsClient.decrypt(request);
            return response.plaintext().asUtf8String();
            
        } catch (KmsException e) {
            throw new RuntimeException("Decryption failed", e);
        }
    }
}
```

### Option 3: Password Hashing (BCrypt)

**Maven Dependencies**:
```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-crypto</artifactId>
    <version>6.2.0</version>
</dependency>
```

**Java Implementation**:
```java
package com.example.security;

import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;

@Service
public class PasswordService {
    
    private final PasswordEncoder passwordEncoder;
    
    public PasswordService() {
        this.passwordEncoder = new BCryptPasswordEncoder(12);  // Strength 12
    }
    
    public String hashPassword(String plainPassword) {
        return passwordEncoder.encode(plainPassword);
    }
    
    public boolean verifyPassword(String plainPassword, String hashedPassword) {
        return passwordEncoder.matches(plainPassword, hashedPassword);
    }
}
```

**Usage**:
```java
@Service
public class UserService {
    
    private final PasswordService passwordService;
    private final UserRepository userRepository;
    
    public void registerUser(String email, String password) {
        // Hash password before storing
        String hashedPassword = passwordService.hashPassword(password);
        
        User user = new User();
        user.setEmail(email);
        user.setPassword(hashedPassword);  // Store hashed, never plain text
        
        userRepository.save(user);
    }
    
    public boolean authenticate(String email, String password) {
        User user = userRepository.findByEmail(email)
            .orElseThrow(() -> new UserNotFoundException());
        
        // Verify password
        return passwordService.verifyPassword(password, user.getPassword());
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Tradeoffs

| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| **Application-Level Encryption** | Full control, flexible | Performance overhead, key management | Sensitive fields, compliance |
| **Database-Level Encryption (TDE)** | Transparent, no code changes | Less granular, all-or-nothing | Compliance requirement, legacy systems |
| **Field-Level Encryption** | Granular control | Complex, performance impact | High-security requirements |
| **Hashing (Passwords)** | One-way, can't reverse | Can't decrypt, only verify | Passwords, sensitive identifiers |

### Common Pitfalls

**Pitfall 1: Encrypting Passwords Instead of Hashing**
```java
// ❌ BAD: Encrypting passwords (reversible)
String encryptedPassword = encryptionService.encrypt(password);
// Attacker gets key → can decrypt all passwords

// ✅ GOOD: Hashing passwords (one-way)
String hashedPassword = passwordEncoder.encode(password);
// Even with hash, can't get original password
```

**Pitfall 2: Using Weak Encryption**
```java
// ❌ BAD: Weak algorithm
Cipher cipher = Cipher.getInstance("DES");  // 56-bit, broken

// ✅ GOOD: Strong algorithm
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");  // 256-bit, authenticated
```

**Pitfall 3: Storing Encryption Keys in Code**
```java
// ❌ TERRIBLE: Key in code
private static final String ENCRYPTION_KEY = "mySecretKey123";

// ✅ GOOD: Key from KMS
String key = keyManagementService.getKey("data-encryption-key");
```

**Pitfall 4: Not Encrypting Backups**
```bash
# ❌ BAD: Unencrypted backup
pg_dump mydb > backup.sql
# If backup stolen, all data exposed

# ✅ GOOD: Encrypted backup
pg_dump mydb | gpg --encrypt --recipient backup-key > backup.sql.gpg
# Backup encrypted, requires key to decrypt
```

**Pitfall 5: Logging Sensitive Data**
```java
// ❌ BAD: Logging sensitive data
logger.info("User SSN: {}", user.getSsn());

// ✅ GOOD: Never log sensitive data
logger.info("User accessed profile: {}", user.getId());
// SSN never logged
```

### Performance Considerations

**Encryption Overhead**:
- AES-256 encryption: ~1-5ms per operation
- For high-throughput systems: use connection pooling, caching
- Consider encrypting only sensitive fields, not entire records

**Optimization Strategies**:
```java
// Cache decrypted values (with TTL)
private final Map<String, CachedValue> cache = new ConcurrentHashMap<>();

public String getDecryptedValue(String encrypted) {
    CachedValue cached = cache.get(encrypted);
    if (cached != null && !cached.isExpired()) {
        return cached.getValue();
    }
    
    String decrypted = encryptionService.decrypt(encrypted);
    cache.put(encrypted, new CachedValue(decrypted, ttl));
    return decrypted;
}
```

---

## 8️⃣ When NOT to Use This

### Anti-Patterns

**Don't Encrypt**:
1. **Public data**: Product names, public blog posts
2. **Indexed fields**: Encryption breaks database indexes (use searchable encryption if needed)
3. **Computed fields**: Derived values that can be recalculated
4. **Temporary data**: Cache entries, session data (use TTL instead)

**When Plain Text is Acceptable**:
- Public information (product catalog)
- Non-sensitive identifiers (user IDs, order numbers)
- Metadata (timestamps, status flags)
- Data that's already public (public profiles)

**Over-Encryption Warning**:
- Encrypting everything hurts performance
- Adds complexity without security benefit
- Focus encryption on truly sensitive data

---

## 9️⃣ Comparison with Alternatives

### Encryption vs Hashing

| Feature | Encryption | Hashing |
|---------|-----------|---------|
| **Reversible** | ✅ Yes (with key) | ❌ No (one-way) |
| **Use Case** | Data you need to read | Passwords, fingerprints |
| **Key Required** | ✅ Yes | ❌ No |
| **Performance** | Slower | Faster |
| **Example** | Credit cards, SSNs | Passwords, file checksums |

**When to Use Each**:
- **Encryption**: Credit cards (need to charge), SSNs (need to display)
- **Hashing**: Passwords (only verify, never read), file integrity (verify unchanged)

### Application-Level vs Database-Level Encryption

| Feature | Application-Level | Database-Level (TDE) |
|---------|------------------|---------------------|
| **Granularity** | ✅ Field-level | ❌ Entire database |
| **Performance** | ⚠️ Per-field overhead | ✅ Transparent, optimized |
| **Code Changes** | ⚠️ Requires code changes | ✅ No code changes |
| **Flexibility** | ✅ Full control | ❌ Limited options |
| **Key Management** | ✅ Application controls | ⚠️ Database controls |

**When to Choose Each**:
- **Application-Level**: Need field-level control, different encryption per field
- **Database-Level**: Compliance requirement, legacy systems, want transparency

---

## 🔟 Interview Follow-up Questions WITH Answers

### Question 1: "How do you handle encryption key rotation without downtime?"

**Answer**:
Dual-key encryption with gradual migration:

1. **Dual Keys**: Maintain old and new encryption keys simultaneously
2. **Dual Read**: Try decrypting with new key first, fallback to old key
3. **Lazy Re-encryption**: Re-encrypt data with new key when accessed
4. **Background Job**: Gradually re-encrypt all data in background
5. **Cutover**: Once all data re-encrypted, remove old key

```java
public class RotatingEncryptionService {
    private SecretKey currentKey;
    private SecretKey previousKey;
    
    public String decrypt(String ciphertext) {
        // Try current key first
        try {
            return decryptWithKey(ciphertext, currentKey);
        } catch (Exception e) {
            // Fallback to previous key
            return decryptWithKey(ciphertext, previousKey);
        }
    }
    
    public String encrypt(String plaintext) {
        // Always use current key for new data
        return encryptWithKey(plaintext, currentKey);
    }
}
```

### Question 2: "How do you ensure data is encrypted in backups?"

**Answer**:
Multiple layers:

1. **Encrypted Storage**: Backup to encrypted storage (S3 with encryption)
2. **Application-Level**: Encrypt backup files before storage
3. **Database-Level**: Use database encryption features (TDE)
4. **Key Management**: Backup encryption keys separately (never with data)
5. **Verification**: Test restore process regularly

```bash
# Encrypted backup process
pg_dump mydb | \
    gpg --encrypt --recipient backup-key | \
    aws s3 cp - s3://backups/db-backup-$(date +%Y%m%d).sql.gpg \
    --sse aws:kms \
    --sse-kms-key-id backup-kms-key
```

### Question 3: "How do you handle PII in logs and error messages?"

**Answer**:
Multiple protection layers:

1. **Log Masking**: Automatically mask PII in logs
2. **Structured Logging**: Use structured logs, never concatenate PII
3. **Error Sanitization**: Remove PII from exceptions
4. **Access Controls**: Limit who can access logs
5. **Retention Policies**: Delete logs after retention period

```java
public class SafeLogger {
    private static final Pattern SSN_PATTERN = 
        Pattern.compile("\\b\\d{3}-\\d{2}-\\d{4}\\b");
    
    public void log(String message, Object... args) {
        String sanitized = message;
        for (Object arg : args) {
            if (arg instanceof String) {
                sanitized = SSN_PATTERN.matcher(sanitized)
                    .replaceAll("***-**-****");
            }
        }
        logger.info(sanitized);
    }
}
```

### Question 4: "What's the difference between encryption and tokenization?"

**Answer**:

**Encryption**:
- Reversible transformation using a key
- Original data can be recovered
- Use case: Need to read original value (credit card for charging)
- Example: AES-256 encryption

**Tokenization**:
- Replace sensitive data with random token
- Original data stored separately (token vault)
- Use case: Don't need original value (payment processing)
- Example: Replace "4532123456789012" with "tok_abc123xyz"

**When to Use Each**:
- **Encryption**: Need original data (SSN display, credit card charging)
- **Tokenization**: Don't need original (payment tokens, reference numbers)

### Question 5: "How do you comply with GDPR's right to be forgotten?"

**Answer**:
Data deletion with encryption key destruction:

1. **Identify Data**: Find all instances of user's PII
2. **Delete Records**: Remove from databases, backups, logs
3. **Destroy Keys**: If data encrypted, destroy encryption keys (makes data unrecoverable)
4. **Verify Deletion**: Audit to confirm deletion
5. **Documentation**: Log deletion for compliance

```java
public void deleteUserData(Long userId) {
    // 1. Delete user record
    userRepository.deleteById(userId);
    
    // 2. Delete from search indexes
    searchService.deleteUser(userId);
    
    // 3. Delete from backups (mark for deletion)
    backupService.scheduleDeletion(userId);
    
    // 4. If encrypted, destroy keys (makes data unrecoverable)
    keyManagementService.destroyKeysForUser(userId);
    
    // 5. Audit log
    auditLog.log("USER_DATA_DELETED", userId, getCurrentUser().getId());
}
```

---

## 1️⃣1️⃣ One Clean Mental Summary

Data protection is like putting your valuables in a safe (encryption at rest) and transporting them in an armored vehicle (encryption in transit). Sensitive data like passwords are hashed (one-way, can't reverse), while data you need to read later (credit cards, SSNs) is encrypted (reversible with key). Keys are stored in a secure vault (KMS) and rotated periodically. Access is logged (audit trail) and controlled (authorization). In a breach, encrypted data is useless without the keys. Never log sensitive data, always encrypt backups, and use field-level encryption for truly sensitive information. Compliance (GDPR, HIPAA, PCI-DSS) requires encryption, access controls, and audit trails.

---

